Generate AI Videos from Text Prompts with OpenAI Sora 2

https://n8nworkflows.xyz/workflows/generate-ai-videos-from-text-prompts-with-openai-sora-2-9341


# Generate AI Videos from Text Prompts with OpenAI Sora 2

### 1. Workflow Overview

This workflow automates the creation of AI-generated videos from text prompts using the OpenAI Sora 2 video generation API. It takes a user-submitted text prompt, sends it to the Sora 2 model to generate a short video, periodically checks the generation status, and downloads the completed video once ready.

The workflow is logically divided into these main blocks:

- **1.1 Input Reception:** Captures the user’s text prompt via a form trigger to initiate the video creation process.
- **1.2 Video Generation Request:** Sends the prompt to the OpenAI Sora 2 API to start video generation.
- **1.3 Status Polling Loop:** Waits for a defined interval, then checks if the video generation is complete, repeating the cycle as needed.
- **1.4 Video Download:** Once the video is ready, downloads the final MP4 video file.

Supporting notes guide the user on setup, usage, and customization.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** This block handles the initial input by collecting the text prompt for the desired AI video from the user via a form interface. It triggers the workflow to start.
- **Nodes Involved:**  
  - Text Prompt

- **Node Details:**

  - **Text Prompt**
    - Type: Form Trigger  
    - Role: Starts the workflow by receiving user input from a web form.  
    - Configuration:  
      - Form titled “AI video generator”  
      - Single required field “prompt” for the text describing the desired video.  
      - No appended attribution on responses.  
      - Webhook ID set to receive form submissions.  
    - Expressions/Variables: Uses `$json.prompt` to pass the prompt downstream.  
    - Input: External HTTP form submission.  
    - Output: Triggers the next node “Sora 2Video”.  
    - Edge Cases: Missing or empty prompt submission will fail form validation.  
    - Version Requirements: Form Trigger node version 2.2.  
    - No sub-workflow.

---

#### 2.2 Video Generation Request

- **Overview:** Sends the user’s prompt to the OpenAI Sora 2 API to initiate video creation with specified parameters (model, size, duration).
- **Nodes Involved:**  
  - Sora 2Video  
  - Wait for Video

- **Node Details:**

  - **Sora 2Video**
    - Type: HTTP Request  
    - Role: Posts the video generation request to the OpenAI API endpoint `https://api.openai.com/v1/videos`.  
    - Configuration:  
      - Method: POST  
      - Content-Type: multipart/form-data  
      - Body parameters include:  
        - `model`: "sora-2"  
        - `prompt`: set dynamically from `$json.prompt` received from Form Trigger  
        - `size`: "720x1280" (vertical HD resolution)  
        - `seconds`: "4" (video length in seconds)  
      - Authentication: HTTP Header with API key credential named "Your_API_Key".  
    - Expressions: Uses `={{ $json.prompt }}` to inject user prompt.  
    - Input: From “Text Prompt” node.  
    - Output: Passes response JSON containing video generation metadata including video `id`.  
    - Edge Cases:  
      - API auth failures (invalid/missing key)  
      - Network errors or timeouts  
      - API quota limits or model errors  
    - Version Requirements: HTTP Request node version 4.2.  
    - No sub-workflow.

  - **Wait for Video**
    - Type: Wait  
    - Role: Pauses workflow execution for 1 minute before status check.  
    - Configuration:  
      - Duration: 1 minute  
    - Input: From “Sora 2Video” node.  
    - Output: Passes data to “Get Video” node to check status.  
    - Edge Cases: Delay too short/long may cause inefficient polling or delays.  
    - Version Requirements: Wait node version 1.1.

---

#### 2.3 Status Polling Loop

- **Overview:** Periodically queries the OpenAI API to check if the video generation is complete. If not complete, the workflow waits and retries.
- **Nodes Involved:**  
  - Get Video  
  - Check if Video Generation is Complete1  
  - Wait for Video (loop back)

- **Node Details:**

  - **Get Video**
    - Type: HTTP Request  
    - Role: Retrieves the current status of the video generation from OpenAI API endpoint `https://api.openai.com/v1/videos/{{ $json.id }}`.  
    - Configuration:  
      - Method: GET  
      - URL dynamically set with video `id` from previous response.  
      - Authentication: HTTP Header with API key.  
    - Expressions: `=https://api.openai.com/v1/videos/{{ $json.id }}`  
    - Input: From “Wait for Video” node.  
    - Output: Returns JSON with video status and metadata.  
    - Edge Cases:  
      - Invalid video ID  
      - API rate limits  
      - Network errors  
    - Version Requirements: HTTP Request node version 4.2.

  - **Check if Video Generation is Complete1**
    - Type: If (conditional)  
    - Role: Evaluates if the video generation is completed successfully by checking the success flag in the JSON response.  
    - Configuration:  
      - Condition: Checks if `$json.data.successFlag == 1` evaluates to true.  
      - Uses loose type validation to tolerate possible data format variations.  
    - Input: From “Get Video” node.  
    - Output:  
      - True branch proceeds to “Download Video” node.  
      - False branch loops back to “Wait for Video” node to wait and retry status check.  
    - Edge Cases:  
      - Missing or malformed `successFlag` field  
      - Unexpected API response structure  
    - Version Requirements: If node version 2.2.

---

#### 2.4 Video Download

- **Overview:** Downloads the generated video file in MP4 format once the video generation is complete.
- **Nodes Involved:**  
  - Download Video

- **Node Details:**

  - **Download Video**
    - Type: HTTP Request  
    - Role: Downloads the video content from OpenAI API endpoint `https://api.openai.com/v1/videos/{{ $json.id }}/content`.  
    - Configuration:  
      - Method: GET  
      - URL dynamically constructed using video `id`.  
      - Authentication: HTTP Header with API key.  
    - Input: From “Check if Video Generation is Complete1” true branch.  
    - Output: Binary video file data (MP4 format) available for downstream processing or saving.  
    - Edge Cases:  
      - Video file not yet available (race condition)  
      - Network or API errors during download  
    - Version Requirements: HTTP Request node version 4.2.

---

#### 2.5 Sticky Notes and Documentation

- **Overview:** Several sticky notes provide user guidance on workflow setup, usage, and links to tutorials and support.
- **Nodes Involved:**  
  - Sticky Note8  
  - Sticky Note  
  - Sticky Note1  
  - Sticky Note2

- **Node Details:**

  - **Sticky Note8**
    - Type: Sticky Note  
    - Role: Comprehensive setup and usage instructions including obtaining OpenAI API key, parameter reference, workflow logic explanation, and support contact links.  
    - Position: Top-left for easy reference.  
    - Includes links:  
      - [YouTube Channel](https://www.youtube.com/@AIBIZElevate77)  
      - [OpenAI API Key Dashboard](https://openai.com/api/)  
      - [Sora 2 API Documentation](https://platform.openai.com/docs/models/sora-2)  
      - [Appointment Booking](https://aibizelevate.com/)  
      - [LinkedIn Profile](https://www.linkedin.com/in/barbora-svobodova-461b92285/)

  - **Sticky Note**
    - Role: Explains the purpose of the trigger node (Text Prompt).  

  - **Sticky Note1**
    - Role: Explains video creation request and status checking loop.

  - **Sticky Note2**
    - Role: Explains the video download step.

---

### 3. Summary Table

| Node Name                           | Node Type          | Functional Role                                  | Input Node(s)          | Output Node(s)                 | Sticky Note                                                                                                      |
|-----------------------------------|--------------------|-------------------------------------------------|------------------------|-------------------------------|-----------------------------------------------------------------------------------------------------------------|
| Text Prompt                       | Form Trigger       | Receives user prompt to start video generation   | -                      | Sora 2Video                   | ## 1. Trigger Node Fill your video prompt in the form and sumbit it to start the automation.                     |
| Sora 2Video                      | HTTP Request       | Sends video generation request to OpenAI Sora 2 | Text Prompt            | Wait for Video                | ## 2. Create Video and Check Status When the video creation request is submitted, the automation checks the video completion status in regular intervals. |
| Wait for Video                   | Wait               | Waits 1 minute before checking video status      | Sora 2Video, Check if Video Generation is Complete1 (false branch) | Get Video                     | ## 2. Create Video and Check Status When the video creation request is submitted, the automation checks the video completion status in regular intervals. |
| Get Video                       | HTTP Request       | Fetches current video generation status          | Wait for Video          | Check if Video Generation is Complete1 | ## 2. Create Video and Check Status When the video creation request is submitted, the automation checks the video completion status in regular intervals. |
| Check if Video Generation is Complete1 | If                 | Checks if video generation succeeded             | Get Video               | Download Video (true), Wait for Video (false) | ## 2. Create Video and Check Status When the video creation request is submitted, the automation checks the video completion status in regular intervals. |
| Download Video                  | HTTP Request       | Downloads the completed video in MP4 format      | Check if Video Generation is Complete1 (true) | -                             | ## Download Video Download the completed video in mp4 format.                                                  |
| Sticky Note8                   | Sticky Note        | Setup instructions, usage explanation, support links | -                      | -                             | For more tutorials visit our [Youtube](https://www.youtube.com/@AIBIZElevate77) ... [LinkedIn](https://www.linkedin.com/in/barbora-svobodova-461b92285/) |
| Sticky Note                    | Sticky Note        | Describes the trigger node                        | -                      | -                             | ## 1. Trigger Node Fill your video prompt in the form and sumbit it to start the automation.                     |
| Sticky Note1                   | Sticky Note        | Describes video creation and status checking     | -                      | -                             | ## 2. Create Video and Check Status When the video creation request is submitted, the automation checks the video completion status in regular intervals. |
| Sticky Note2                   | Sticky Note        | Describes video download step                      | -                      | -                             | ## Download Video Download the completed video in mp4 format.                                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create “Text Prompt” node**  
   - Type: Form Trigger  
   - Configuration:  
     - Form Title: "AI video generator"  
     - Add one required field named `prompt` (text input)  
     - Disable "Append Attribution"  
   - This node will receive a webhook URL for form submission.  

2. **Create “Sora 2Video” node**  
   - Type: HTTP Request  
   - Set method to POST  
   - URL: `https://api.openai.com/v1/videos`  
   - Content-Type: multipart/form-data  
   - Body parameters:  
     - model: "sora-2"  
     - prompt: set expression to `{{$json.prompt}}` from trigger node  
     - size: "720x1280"  
     - seconds: "4"  
   - Authentication: HTTP Header Auth  
   - Credential: Create or select credential with your OpenAI API key (name it e.g., “Your_API_Key”)  
   - Connect output of “Text Prompt” node to input of “Sora 2Video”.  

3. **Create “Wait for Video” node**  
   - Type: Wait  
   - Set unit to “minutes”  
   - Set amount to 1  
   - Connect output of “Sora 2Video” to input of “Wait for Video”.

4. **Create “Get Video” node**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: Set expression to `https://api.openai.com/v1/videos/{{$json.id}}`  
   - Authentication: HTTP Header Auth (use same OpenAI API key credential)  
   - Connect output of “Wait for Video” to input of “Get Video”.

5. **Create “Check if Video Generation is Complete1” node**  
   - Type: If  
   - Condition: Check if expression `$json.data.successFlag == 1` is true  
   - Use loose type validation  
   - Connect output of “Get Video” to input of this node.  

6. **Create “Download Video” node**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: Expression `https://api.openai.com/v1/videos/{{$json.id}}/content`  
   - Authentication: HTTP Header Auth (same OpenAI API Key)  
   - Connect “true” output of the If node to “Download Video”.

7. **Close the polling loop**  
   - Connect “false” output of the If node back to “Wait for Video” to enable repeated status checks.

8. **Add Sticky Notes** (optional but recommended for clarity)  
   - Add sticky notes with setup instructions, explanations of each block, and useful links such as:  
     - OpenAI API key dashboard  
     - Sora 2 API documentation  
     - YouTube tutorials and support contacts  

9. **Save and activate the workflow**  
   - Ensure the API key credential is configured correctly in all HTTP Request nodes.  
   - Test the webhook URL generated by the Form Trigger node by submitting a sample prompt to initiate video creation.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                               | Context or Link                                                                                  |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| For more tutorials visit our [YouTube channel](https://www.youtube.com/@AIBIZElevate77)                                                                                                                                                                                                                                                                                                                   | Video tutorials for n8n and AI automation                                                     |
| Obtain your OpenAI API key from the [OpenAI account dashboard](https://openai.com/api/)                                                                                                                                                                                                                                                                                                                    | API credential setup                                                                           |
| Adjust video parameters such as size and length referring to Sora 2 API [documentation](https://platform.openai.com/docs/models/sora-2)                                                                                                                                                                                                                                                                   | Official OpenAI Sora 2 model docs                                                             |
| Need help or want to customize? Book an appointment at [aibizelevate.com](https://aibizelevate.com/) or connect on [LinkedIn](https://www.linkedin.com/in/barbora-svobodova-461b92285/)                                                                                                                                                                                                                   | Support and customization services                                                           |

---

**Disclaimer:**  
The text provided is exclusively from an automated workflow created using n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.