Turn Google Sheets Ideas into AI Videos with GPT-4o and Fal.AI Veo 3

https://n8nworkflows.xyz/workflows/turn-google-sheets-ideas-into-ai-videos-with-gpt-4o-and-fal-ai-veo-3-4881


# Turn Google Sheets Ideas into AI Videos with GPT-4o and Fal.AI Veo 3

### 1. Workflow Overview

This workflow automates the creation of short AI-generated videos using ideas provided in a Google Sheets spreadsheet. When a new row is added with an idea, video ratio, and audio preference, the workflow generates a creative prompt via GPT-4o, submits a video generation request to Fal.AI’s Veo 3 model, monitors the video rendering status asynchronously, and finally updates the Google Sheet with the video URL and prompt used.

The workflow is logically divided into four main blocks:

- **1.1 Trigger & Prompt Generation:** Detects new rows in Google Sheets and generates a detailed video prompt using OpenAI’s GPT-4o model, tailored for Fal.AI’s Veo 3 video generation.
- **1.2 Prepare & Submit Video Request:** Sets necessary parameters and sends the prompt to Fal.AI’s API to request video creation.
- **1.3 Monitor Video Status:** Periodically checks the video generation status until completion.
- **1.4 Retrieve & Update Results:** Fetches the final video URL from Fal.AI and updates the Google Sheet row with the generated video link and prompt.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger & Prompt Generation

**Overview:**  
This block listens for new rows added in a specific Google Sheets document. Upon detecting a new idea entry, it crafts a rich, immersive prompt optimized for Veo 3 video generation using GPT-4o.

**Nodes Involved:**  
- Google Sheets Trigger  
- Generate prompt for Veo 3 model  
- Sticky Note1 (documentation)

**Node Details:**

- **Google Sheets Trigger**  
  - *Type:* Trigger node  
  - *Role:* Monitors a Google Sheet for newly added rows every minute.  
  - *Configuration:* Uses OAuth2 credentials linked to a Google account with access to the target spreadsheet and sheet (gid=0). Triggers on "rowAdded" event.  
  - *Expressions:* Reads fields like `Idea`, `Ratio`, and `Audio` from the new row.  
  - *Connections:* Output flows into the "Generate prompt for Veo 3 model" node.  
  - *Edge Cases:* Possible auth failures, API quota limits, or misconfigured spreadsheet ID/sheet name.

- **Generate prompt for Veo 3 model**  
  - *Type:* LangChain OpenAI node  
  - *Role:* Uses OpenAI’s GPT-4o chat model to generate a detailed video prompt based on the "Idea" field from Google Sheets.  
  - *Configuration:* Model set to "chatgpt-4o-latest". Message instructs the model to act as a creative prompt engineer for Veo 3, constructing rich, detailed prompts including setting, lighting, mood, style, and composition.  
  - *Expressions:* Input includes `{{ $json.Idea }}` from the Google Sheets trigger.  
  - *Connections:* Output flows to "Set variables for Video generation" node.  
  - *Edge Cases:* OpenAI API key invalid, rate limits, or unexpected input format causing prompt generation failure.

- **Sticky Note1**  
  - Provides a high-level explanation of this stage for users.

---

#### 2.2 Prepare & Submit Video Request

**Overview:**  
This block prepares and assigns necessary variables such as the generated prompt, video aspect ratio, and audio preference. It then submits a video generation request to Fal.AI’s Veo 3 API.

**Nodes Involved:**  
- Set variables for Video generation  
- Submit Request to generate video  
- Sticky Note2 (documentation)

**Node Details:**

- **Set variables for Video generation**  
  - *Type:* Set node  
  - *Role:* Maps output from prompt generation and Google Sheets columns to variables used in the video request.  
  - *Configuration:*  
    - `prompt` set to the content of the generated prompt message.  
    - `ratio` mapped from Google Sheets "Ratio" column.  
    - `audio` mapped from Google Sheets "Audio" column (expects string "true" or "false").  
  - *Connections:* Output flows to "Submit Request to generate video".  
  - *Edge Cases:* Mismatched field names or empty values could break request formation.

- **Submit Request to generate video**  
  - *Type:* HTTP Request node  
  - *Role:* Sends POST request to Fal.AI Veo 3 endpoint to start video generation.  
  - *Configuration:*  
    - URL: `https://queue.fal.run/fal-ai/veo3`  
    - Headers: `Content-Type: application/json`, `Authorization: Key <YOUR_API_KEY>` (replace with actual Fal.AI API key).  
    - Body JSON includes prompt, aspect_ratio, enhance_prompt (true), and generate_audio.  
    - Uses expressions to insert variables from previous node.  
  - *Connections:* Output flows to "Check video status".  
  - *Edge Cases:* Authorization failure, malformed request, network errors, or API downtime.

- **Sticky Note2**  
  - Describes this stage’s purpose to prepare and submit the video request.

---

#### 2.3 Monitor Video Status

**Overview:**  
This block continuously polls Fal.AI’s API to check if the video generation is complete, waiting 5 seconds between checks as needed.

**Nodes Involved:**  
- Check video status  
- Check if video is ready (IF node)  
- Wait 5s  
- Sticky Note3 (documentation)

**Node Details:**

- **Check video status**  
  - *Type:* HTTP Request node  
  - *Role:* Queries Fal.AI API for the status of the video generation request using request_id from the previous response.  
  - *Configuration:*  
    - URL template: `https://queue.fal.run/fal-ai/veo3/requests/{{ $json.request_id }}/status`  
    - Header includes Fal.AI API key for authorization.  
  - *Connections:* Output flows to "Check if video is ready".  
  - *Edge Cases:* API failures, invalid request_id, or network issues.

- **Check if video is ready**  
  - *Type:* IF node  
  - *Role:* Evaluates if the response status equals "COMPLETED".  
  - *Configuration:* Checks `$json.status == "COMPLETED"`.  
  - *Connections:*  
    - If true: flows to "Get video url" node.  
    - If false: flows to "Wait 5s" node to pause before next status check.  
  - *Edge Cases:* Unexpected status values, JSON parsing errors.

- **Wait 5s**  
  - *Type:* Wait node  
  - *Role:* Pauses workflow execution for 5 seconds to prevent excessive API calls.  
  - *Configuration:* Wait time set to 5 seconds.  
  - *Connections:* Loops back to "Check video status" to re-check status after waiting.  
  - *Edge Cases:* Execution time limits, node interruptions.

- **Sticky Note3**  
  - Explains this polling and waiting mechanism for monitoring video readiness.

---

#### 2.4 Retrieve & Update Results

**Overview:**  
Once the video is ready, this block fetches the video URL and updates the original Google Sheets row with both the generated prompt and the video link.

**Nodes Involved:**  
- Get video url  
- Update sheet with video url and prompt used in Veo3  
- Sticky Note4 (documentation)

**Node Details:**

- **Get video url**  
  - *Type:* HTTP Request node  
  - *Role:* Retrieves the video URL from Fal.AI using the `response_url` provided in the status check response.  
  - *Configuration:*  
    - URL uses `$json.response_url` from previous node.  
    - Authorization header includes Fal.AI API key.  
  - *Connections:* Output flows to "Update sheet with video url and prompt used in Veo3".  
  - *Edge Cases:* Expired or invalid URL, auth issues, malformed response.

- **Update sheet with video url and prompt used in Veo3**  
  - *Type:* Google Sheets node  
  - *Role:* Updates corresponding row in Google Sheets with the newly generated video URL and the prompt text.  
  - *Configuration:*  
    - Matches row by "Idea" column.  
    - Updates columns "Video Generated" with video URL and "Prompt Generated" with prompt text.  
    - Uses OAuth2 credentials with write access.  
  - *Connections:* Terminal node (no outputs).  
  - *Edge Cases:* Row matching failure, permission issues, API limits.

- **Sticky Note4**  
  - Describes updating Google Sheets with final output data.

---

### 3. Summary Table

| Node Name                              | Node Type                               | Functional Role                               | Input Node(s)               | Output Node(s)                          | Sticky Note                                                  |
|--------------------------------------|---------------------------------------|-----------------------------------------------|-----------------------------|----------------------------------------|--------------------------------------------------------------|
| Google Sheets Trigger                 | Google Sheets Trigger                  | Detect new rows added in Google Sheets        | —                           | Generate prompt for Veo 3 model         | Sticky Note1: ## Stage 1: Trigger & Prompt Generation        |
| Generate prompt for Veo 3 model       | LangChain OpenAI                      | Generate detailed AI video prompt              | Google Sheets Trigger       | Set variables for Video generation      |                                                              |
| Set variables for Video generation    | Set                                   | Map prompt, ratio, and audio variables         | Generate prompt for Veo 3 model | Submit Request to generate video       | Sticky Note2: ## Stage 2: Prepare & Submit Video Request     |
| Submit Request to generate video      | HTTP Request                         | Send video generation request to Fal.AI       | Set variables for Video generation | Check video status                     |                                                              |
| Check video status                    | HTTP Request                         | Poll Fal.AI API to get video generation status | Submit Request to generate video | Check if video is ready                 | Sticky Note3: ## Stage 3: Monitor Video Status               |
| Check if video is ready               | IF                                    | Evaluate if video generation is complete       | Check video status          | Get video url (if yes), Wait 5s (if no) |                                                              |
| Wait 5s                              | Wait                                  | Pause 5 seconds before rechecking status       | Check if video is ready (no) | Check video status                     |                                                              |
| Get video url                        | HTTP Request                         | Retrieve final video URL from Fal.AI API       | Check if video is ready (yes) | Update sheet with video url and prompt | Sticky Note4: ## Stage 4: Retrieve & Update Results          |
| Update sheet with video url and prompt used in Veo3 | Google Sheets                      | Update Google Sheet row with video URL and prompt | Get video url               | —                                      |                                                              |
| Sticky Note1                        | Sticky Note                          | Documentation for Trigger & Prompt Generation  | —                           | —                                      | See content in Sticky Note1 section                           |
| Sticky Note2                        | Sticky Note                          | Documentation for Prepare & Submit Video Request | —                           | —                                      | See content in Sticky Note2 section                           |
| Sticky Note3                        | Sticky Note                          | Documentation for Monitor Video Status          | —                           | —                                      | See content in Sticky Note3 section                           |
| Sticky Note4                        | Sticky Note                          | Documentation for Retrieve & Update Results     | —                           | —                                      | See content in Sticky Note4 section                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Sheets Trigger node:**  
   - Type: Google Sheets Trigger  
   - Trigger on event: "rowAdded"  
   - Poll every minute  
   - Set spreadsheet ID and sheet name (gid=0) corresponding to your Google Sheet  
   - Connect Google Sheets OAuth2 credentials with access to the sheet

2. **Create OpenAI LangChain node:**  
   - Name: Generate prompt for Veo 3 model  
   - Type: LangChain OpenAI  
   - Model: chatgpt-4o-latest  
   - Message template:  
     - Assistant role message instructing to generate a detailed Veo 3 prompt with rich visual details  
     - User message with input: `={{ $json.Idea }}` from Google Sheets Trigger  
   - Connect OpenAI credentials with valid API key  
   - Connect output from Google Sheets Trigger to this node

3. **Create Set node:**  
   - Name: Set variables for Video generation  
   - Create variables:  
     - `prompt`: `={{ $json.message.content }}` (from OpenAI response)  
     - `ratio`: `={{ $('Google Sheets Trigger').item.json.Ratio }}`  
     - `audio`: `={{ $('Google Sheets Trigger').item.json.Audio }}` (ensure true/false strings)  
   - Connect output from OpenAI node to this node

4. **Create HTTP Request node to submit video:**  
   - Name: Submit Request to generate video  
   - Method: POST  
   - URL: `https://queue.fal.run/fal-ai/veo3`  
   - Headers:  
     - Content-Type: application/json  
     - Authorization: `Key <YOUR_API_KEY>` (replace with your Fal.AI API key)  
   - Body (JSON):  
     ```json
     {
       "prompt": "{{ $json.prompt }}",
       "aspect_ratio": "{{ $json.ratio }}",
       "enhance_prompt": true,
       "generate_audio": {{ $json.audio }}
     }
     ```  
   - Connect output from Set node to this node

5. **Create HTTP Request node to check video status:**  
   - Name: Check video status  
   - Method: GET  
   - URL: `https://queue.fal.run/fal-ai/veo3/requests/{{ $json.request_id }}/status`  
   - Header: Authorization with Fal.AI API key  
   - Connect output from Submit Request node to this node

6. **Create IF node to check completion:**  
   - Name: Check if video is ready  
   - Condition: `$json.status === "COMPLETED"`  
   - Connect output from Check video status to this node

7. **Create Wait node:**  
   - Name: Wait 5s  
   - Duration: 5 seconds  
   - Connect the false output (not completed) from IF node to this Wait node  
   - Connect Wait node output back to Check video status node to loop

8. **Create HTTP Request node to get video URL:**  
   - Name: Get video url  
   - Method: GET  
   - URL: `={{ $json.response_url }}` (from status API response)  
   - Header: Authorization with Fal.AI API key  
   - Connect true output (completed) from IF node to this node

9. **Create Google Sheets node to update sheet:**  
   - Name: Update sheet with video url and prompt used in Veo3  
   - Operation: Update  
   - Spreadsheet ID and sheet name same as trigger  
   - Matching column: "Idea"  
   - Columns to update:  
     - "Video Generated" with `={{ $json.video.url }}` from Get video url node  
     - "Prompt Generated" with `={{ $('Generate prompt for Veo 3 model').item.json.message.content }}`  
   - Connect output from Get video url node to this node  
   - Use Google Sheets OAuth2 credentials with write permissions

10. **Optional:** Add sticky notes for documentation at each stage to guide users.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                  | Context or Link                                                                                          |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| This workflow enables easy creation of 8-second AI videos from Google Sheets entries. Fal.AI charges are approximately $0.50 per second of video, $0.75 per second with audio. Execution times vary from 3 to 15 minutes depending on audio inclusion.                                                                                                                                        | Pricing and performance notes from Sticky Note attached in the workflow                                 |
| Ensure Google Sheets API is enabled in your Google Cloud project, and OAuth2 credentials are properly configured for both reading and writing.                                                                                                                                                                                                                                               | Setup instructions from Sticky Note1 and Sticky Note4                                                  |
| OpenAI API key must be valid and able to access GPT-4o model.                                                                                                                                                                                                                                                                                                                               | Credentials info from node configuration                                                               |
| Fal.AI API key must be inserted in all relevant HTTP Request nodes (`Submit Request to generate video`, `Check video status`, `Get video url`) replacing `<YOUR_API_KEY>`.                                                                                                                                                                                                                     | API key requirements detailed in Sticky Note and HTTP Request nodes                                    |
| If the "Audio" column in Google Sheets is Boolean type and causes errors, enter it explicitly as string `"true"` or `"false"` in the sheet (e.g., `="true"`).                                                                                                                                                                                                                                 | Troubleshooting note in Sticky Note                                                                   |
| For questions or support, contact max@nervoai.com.                                                                                                                                                                                                                                                                                                                                            | Contact info from Sticky Note                                                                          |
| Fal.AI Veo 3 model documentation and API details: https://fal.ai/docs/api/veo3                                                                                                                                                                                                                                                                                                                  | External resource link                                                                                  |
| Google Sheets API documentation: https://developers.google.com/sheets/api                                                                                                                                                                                                                                                                                                                     | External resource link                                                                                  |
| OpenAI API documentation: https://platform.openai.com/docs/api-reference                                                                                                                                                                                                                                                                                                                     | External resource link                                                                                  |

---

**Disclaimer:**  
The text analyzed and documented is derived exclusively from an automated workflow created in n8n, an integration and automation platform. This process strictly adheres to applicable content policies and contains no illegal, offensive, or protected material. All manipulated data is legal and publicly accessible.