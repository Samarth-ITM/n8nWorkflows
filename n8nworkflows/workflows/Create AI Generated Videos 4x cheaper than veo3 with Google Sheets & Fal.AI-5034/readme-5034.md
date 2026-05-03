Create AI Generated Videos 4x cheaper than veo3 with Google Sheets & Fal.AI

https://n8nworkflows.xyz/workflows/create-ai-generated-videos-4x-cheaper-than-veo3-with-google-sheets---fal-ai-5034


# Create AI Generated Videos 4x cheaper than veo3 with Google Sheets & Fal.AI

### 1. Workflow Overview

This workflow automates the creation of AI-generated videos using the Fal.AI Kling 2.1 video model, triggered by new rows added to a Google Sheet. It is designed for users who want to produce short, 5-second videos cost-effectively (advertised as 4x cheaper than Veo3) by leveraging Google Sheets as an easy input interface and Fal.AI as the video generation backend.

The workflow consists of four main logical blocks:

- **1.1 Trigger & Prompt Generation:** Detects new rows in a Google Sheet and generates a detailed video prompt using OpenAI's GPT-4.1 model tailored for Kling 2.1 video generation.
- **1.2 Prepare & Submit Video Request:** Sets up necessary parameters and submits the video generation request to Fal.AI.
- **1.3 Monitor Video Status:** Polls Fal.AI periodically to check if the video generation is completed, with a wait interval between checks.
- **1.4 Retrieve & Update Results:** Once the video is ready, retrieves the video URL and updates the original Google Sheet row with the generated prompt and video link.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger & Prompt Generation

**Overview:**  
This block initiates the workflow when a new row is added to the specified Google Sheet. It then generates a rich, descriptive prompt for Fal.AI’s Kling 2.1 video model based on the user’s input idea.

**Nodes Involved:**  
- Google Sheets Trigger  
- Generate prompt for Kling 2.1 model

**Node Details:**

- **Google Sheets Trigger**  
  - Type: Trigger node for Google Sheets  
  - Configuration: Watches for new rows added to the sheet with ID `1VGhoJU-qHzav53pTUSPtX6PNVPrRp9e38E34AmHoWvg`, specifically the first sheet (gid=0). Polls every minute.  
  - Credentials: Uses OAuth2 credentials for Google Sheets API.  
  - Inputs: None (trigger)  
  - Outputs: Emits new row data containing columns like Idea, Ratio, Audio, etc.  
  - Potential issues: API quota limits, authentication errors, sheet structure changes.

- **Generate prompt for Kling 2.1 model**  
  - Type: OpenAI (LangChain) node  
  - Configuration: Uses GPT-4.1-nano-2025-04-14 model.  
  - Messages: System prompt instructs to create a detailed, sensory-rich prompt based on the `Idea` field from the Google Sheet row.  
  - Credentials: OpenAI API key required.  
  - Inputs: Receives JSON with the `Idea` field from trigger.  
  - Outputs: Generates a text prompt that will guide video creation.  
  - Edge cases: API rate limits, model response errors, malformed input causing poor prompt generation.

---

#### 1.2 Prepare & Submit Video Request

**Overview:**  
This block sets variables needed for video creation, including the prompt and aspect ratio, then submits the video generation request to Fal.AI’s API.

**Nodes Involved:**  
- Set variables for Video generation  
- Submit Request to generate video

**Node Details:**

- **Set variables for Video generation**  
  - Type: Set node  
  - Configuration: Assigns two variables:  
    - `prompt`: extracted from the AI-generated prompt content.  
    - `ratio`: extracted from the original Google Sheet row's Ratio column.  
  - Inputs: From prompt generation node  
  - Outputs: Passes these variables downstream for the API request.  
  - Edge cases: Missing or malformed prompt or ratio values.

- **Submit Request to generate video**  
  - Type: HTTP Request node  
  - Configuration: POST request to `https://queue.fal.run/fal-ai/kling-video/v2.1/master/text-to-video`  
  - Body: JSON containing the `prompt` and fixed `"aspect_ratio": "9:16"` (Note: ratio from variables is assigned but fixed here)  
  - Headers: Content-Type application/json, Authorization with Fal.AI API key (`Key <YOUR_API_KEY>`)  
  - Inputs: Receives prompt and ratio variables  
  - Outputs: Response contains a `request_id` for status tracking  
  - Edge cases: HTTP errors, invalid API key, API rate limits, malformed prompt causing request rejection.

---

#### 1.3 Monitor Video Status

**Overview:**  
This block continuously checks the status of the video request by polling Fal.AI every 5 seconds until the video processing is completed.

**Nodes Involved:**  
- Check video status  
- Check if video is ready (IF node)  
- Wait 5s

**Node Details:**

- **Check video status**  
  - Type: HTTP Request node  
  - Configuration: GET request to `https://queue.fal.run/fal-ai/kling-video/requests/{{ $json.request_id }}/status`  
  - Header: Authorization with Fal.AI API key  
  - Inputs: Receives `request_id` from previous request node  
  - Outputs: JSON with a `status` field (e.g., "COMPLETED", "PENDING")  
  - Edge cases: Network errors, invalid request_id, API limits.

- **Check if video is ready**  
  - Type: IF node  
  - Configuration: Checks if `status` equals `"COMPLETED"`  
  - Inputs: Output of Check video status node  
  - Outputs:  
    - True branch: proceed to get video URL  
    - False branch: wait and poll again  
  - Edge cases: Unexpected status values, JSON parsing errors.

- **Wait 5s**  
  - Type: Wait node  
  - Configuration: Waits 5 seconds before next status check to avoid excessive polling  
  - Inputs: False branch of IF node  
  - Outputs: Loops back to Check video status node  
  - Edge cases: Workflow timeouts if running too long, node execution delays.

---

#### 1.4 Retrieve & Update Results

**Overview:**  
Once the video generation is complete, this block retrieves the video URL from Fal.AI and updates the corresponding row in Google Sheets with both the prompt and the video link.

**Nodes Involved:**  
- Get video url  
- Update sheet with video url and prompt used in Kling 2.1

**Node Details:**

- **Get video url**  
  - Type: HTTP Request node  
  - Configuration: GET request to the video URL provided by the Fal.AI response (`$json.response_url`)  
  - Header: Authorization with Fal.AI API key  
  - Inputs: True branch of IF node (video ready)  
  - Outputs: Contains video URL inside JSON response  
  - Edge cases: Expired or invalid URL, authorization failure.

- **Update sheet with video url and prompt used in Kling 2.1**  
  - Type: Google Sheets node  
  - Configuration: Update operation on the same Google Sheet (ID `1VGhoJU-qHzav53pTUSPtX6PNVPrRp9e38E34AmHoWvg`) in gid=0.  
  - Matching column: "Idea" to find the row.  
  - Updates columns:  
    - "Video Generated" with the video URL from Fal.AI  
    - "Prompt Generated" with the prompt generated by OpenAI  
  - Credentials: Uses Google Sheets OAuth2 credentials  
  - Inputs: Video URL node output and original prompt  
  - Edge cases: Row matching failures, API write errors, credentials expiration.

---

### 3. Summary Table

| Node Name                              | Node Type                 | Functional Role                                      | Input Node(s)                      | Output Node(s)                            | Sticky Note                                                                                                          |
|--------------------------------------|---------------------------|-----------------------------------------------------|----------------------------------|------------------------------------------|----------------------------------------------------------------------------------------------------------------------|
| Google Sheets Trigger                 | Google Sheets Trigger     | Triggers workflow on new row added                  | None                             | Generate prompt for Kling 2.1 model      | ## Stage 1: Trigger & Prompt Generation Starts the workflow when a new row is added in Google Sheets, then when the workflow is actioned, it generates a prompt for the Kling 2.1 video model based on the sheet data. |
| Generate prompt for Kling 2.1 model  | OpenAI Node               | Generates detailed video prompt from input idea    | Google Sheets Trigger            | Set variables for Video generation       |                                                                                                                      |
| Set variables for Video generation   | Set Node                  | Prepares variables (prompt, ratio) for video API    | Generate prompt for Kling 2.1 model | Submit Request to generate video         | ## Stage 2: Prepare & Submit Video Request Sets up all necessary variables for video creation and sends a request to Fal.AI to generate the video. |
| Submit Request to generate video     | HTTP Request              | Sends video generation request to Fal.AI            | Set variables for Video generation | Check video status                       |                                                                                                                      |
| Check video status                   | HTTP Request              | Polls Fal.AI to check video generation status       | Submit Request to generate video | Check if video is ready                   | ## Stage 3: Monitor Video Status Checks the status of the video generation. Waits and repeatedly checks every 5 seconds until the video is ready. |
| Check if video is ready              | IF Node                   | Branches logic based on video generation completion | Check video status               | Get video url (true), Wait 5s (false)   |                                                                                                                      |
| Wait 5s                             | Wait Node                 | Delays before next status check                      | Check if video is ready (false) | Check video status                        |                                                                                                                      |
| Get video url                       | HTTP Request              | Retrieves the final video URL                         | Check if video is ready (true)  | Update sheet with video url and prompt   | ## Stage 4: Retrieve & Update Results Gets the final video URL and updates the Google Sheet with the video link and the prompt used. |
| Update sheet with video url and prompt used in Kling 2.1 | Google Sheets           | Updates Google Sheet row with prompt and video URL  | Get video url                   | None                                     |                                                                                                                      |
| Sticky Note                         | Sticky Note               | Documentation                                         | None                           | None                                     | ## Create AI Generated Videos 4x cheaper than veo3 with Google Sheets & Fal.AI ... full instructions and troubleshooting notes. |
| Sticky Note1                        | Sticky Note               | Documentation                                         | None                           | None                                     | ## Stage 1: Trigger & Prompt Generation Starts the workflow when a new row is added in Google Sheets, then when the workflow is actioned, it generates a prompt for the Kling 2.1 video model based on the sheet data. |
| Sticky Note2                        | Sticky Note               | Documentation                                         | None                           | None                                     | ## Stage 2: Prepare & Submit Video Request Sets up all necessary variables for video creation and sends a request to Fal.AI to generate the video. |
| Sticky Note3                        | Sticky Note               | Documentation                                         | None                           | None                                     | ## Stage 3: Monitor Video Status Checks the status of the video generation. Waits and repeatedly checks every 5 seconds until the video is ready. |
| Sticky Note4                        | Sticky Note               | Documentation                                         | None                           | None                                     | ## Stage 4: Retrieve & Update Results Gets the final video URL and updates the Google Sheet with the video link and the prompt used. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Sheets Trigger node:**  
   - Type: Google Sheets Trigger  
   - Event: Row Added  
   - Spreadsheet ID: `1VGhoJU-qHzav53pTUSPtX6PNVPrRp9e38E34AmHoWvg` (or your own)  
   - Sheet GID: `0`  
   - Poll interval: Every minute  
   - Credentials: Add OAuth2 credentials for Google Sheets  
   - This node triggers the workflow for each new row.

2. **Add OpenAI node to generate prompt:**  
   - Type: OpenAI (LangChain)  
   - Model: GPT-4.1-nano-2025-04-14  
   - Messages:  
     - System prompt instructing to create a detailed prompt for Kling 2.1 video generation from the input idea.  
     - User prompt: `={{ $json.Idea }}` from the Google Sheets Trigger.  
   - Credentials: OpenAI API key  
   - Connect Google Sheets Trigger → Generate prompt node.

3. **Set node to define video generation variables:**  
   - Type: Set  
   - Assign:  
     - `prompt` = `={{ $json.message.content }}` (output from OpenAI node)  
     - `ratio` = `={{ $('Google Sheets Trigger').item.json.Ratio }}` (from sheet)  
   - Connect Generate prompt → Set variables.

4. **HTTP Request node to submit video request to Fal.AI:**  
   - Type: HTTP Request (POST)  
   - URL: `https://queue.fal.run/fal-ai/kling-video/v2.1/master/text-to-video`  
   - Headers:  
     - Content-Type: application/json  
     - Authorization: `Key <YOUR_API_KEY>` (replace with your Fal.AI API key)  
   - Body (JSON):  
     ```json
     {
       "prompt": "{{ $json.prompt }}",
       "aspect_ratio": "9:16"
     }
     ```  
   - Connect Set variables → Submit Request node.

5. **HTTP Request node to check video status:**  
   - Type: HTTP Request (GET)  
   - URL: `https://queue.fal.run/fal-ai/kling-video/requests/{{ $json.request_id }}/status`  
   - Header: Authorization with Fal.AI API key  
   - Connect Submit Request → Check video status node.

6. **IF node to check if status is 'COMPLETED':**  
   - Condition: `$json.status` equals `COMPLETED` (case sensitive)  
   - True branch: proceed to get video URL  
   - False branch: wait then retry

7. **Wait node:**  
   - Wait 5 seconds  
   - Connect IF node (False) → Wait node → Check video status node (loop).

8. **HTTP Request node to get video URL:**  
   - Type: HTTP Request (GET)  
   - URL: `={{ $json.response_url }}` (from status check response)  
   - Header: Authorization with Fal.AI API key  
   - Connect IF node (True) → Get video URL node.

9. **Google Sheets node to update row:**  
   - Operation: Update  
   - Spreadsheet ID and sheet same as trigger node  
   - Matching column: "Idea"  
   - Update columns:  
     - "Prompt Generated" = `={{ $('Generate prompt for Kling 2.1 model').item.json.message.content }}`  
     - "Video Generated" = `={{ $json.video.url }}` (from Get video URL node)  
   - Credentials: Google Sheets OAuth2  
   - Connect Get video URL → Update sheet node.

10. **Add Sticky Notes for documentation:**  
    - Add sticky notes for each stage with the content provided to explain stages and usage.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              | Context or Link                                                                                                     |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------|
| Create AI Generated Videos 4x cheaper than veo3 with Google Sheets & Fal.AI. Easily generate 5-second videos using Fal.AI’s Kling 2.1 model by adding a row to your Google Sheet with your idea, video ratio, and audio preference. Fal.AI pricing: $0.28 per 1s video, $1.40 per 5s video. Typical execution time: 6 minutes. Step-by-step instructions for Google Sheets, OpenAI, Fal.AI API key setup, and troubleshooting included. Troubleshoot "Audio" column boolean issue by entering values as text (`="true"` or `="false"`). Contact: max@nervoai.com | Full workflow description and user instructions embedded in Sticky Note node.                                         |
| Fal.AI API documentation and key management are essential for this workflow. Ensure your API key is valid and replace all placeholder `<YOUR_API_KEY>` with your actual key in all HTTP Request nodes.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        | Fal.AI dashboard and API documentation (external to n8n).                                                         |
| OpenAI GPT-4.1-nano model is used for prompt generation; ensure your API key has access to this model or update accordingly.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        | OpenAI API usage and token management documentation.                                                               |
| Google Sheets must have columns: Idea, Ratio, Prompt Generated, Video Generated. The workflow matches rows by the "Idea" column for updates, so this must be unique or carefully managed.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          | Google Sheets template linked in the workflow description (copy recommended).                                      |
| API rate limits, network errors, or incorrect data formats (e.g., prompt or ratio missing) can cause failures. Implement monitoring or error handling as needed for production environments.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       | General best practices for API integration and error handling.                                                     |

---

**Disclaimer:** The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.