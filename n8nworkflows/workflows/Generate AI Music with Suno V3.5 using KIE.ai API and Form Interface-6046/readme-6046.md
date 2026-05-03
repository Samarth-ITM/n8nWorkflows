Generate AI Music with Suno V3.5 using KIE.ai API and Form Interface

https://n8nworkflows.xyz/workflows/generate-ai-music-with-suno-v3-5-using-kie-ai-api-and-form-interface-6046


# Generate AI Music with Suno V3.5 using KIE.ai API and Form Interface

### 1. Workflow Overview

This workflow automates AI music generation by integrating the KIE.ai API with the Suno V3.5 model via a user-friendly form interface. It targets musicians, content creators, and developers who want to generate custom music tracks programmatically or interactively. The workflow logically divides into these main blocks:

- **1.1 Input Reception and Parameter Collection:** Users submit music generation parameters (prompt, style, title, and API key) through a form trigger node.
- **1.2 API Request Submission:** The workflow sends a POST request to KIE.ai’s music generation endpoint with the user inputs and fixed model parameters.
- **1.3 Polling and Status Monitoring:** It waits and polls the API every 10 seconds to check the status of the music generation task asynchronously.
- **1.4 Result Retrieval and Formatting:** Once the generation completes successfully, it extracts and formats the audio URLs for output.

Supporting this core logic are multiple sticky notes providing detailed instructions, prerequisites, usage steps, and parameter descriptions to guide users.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Parameter Collection

**Overview:**  
This block collects user inputs via a web form interface and triggers the workflow when the form is submitted.

**Nodes Involved:**  
- Submit Music Generation Parameters (Form Trigger)

**Node Details:**  

- **Submit Music Generation Parameters**  
  - Type: Form Trigger  
  - Role: Presents a form to the user to collect music generation parameters and API key; triggers workflow execution on submission.  
  - Configuration:  
    - Form title: "AI music generator"  
    - Fields: `prompt`, `style`, `title`, `api_key`  
    - Form description prompts users to fill in required fields  
  - Key Expressions: Uses submitted form values directly for subsequent API calls.  
  - Input: External user form submission via webhook.  
  - Output: Passes JSON with user inputs downstream.  
  - Edge Cases:  
    - User submits incomplete or invalid data (no internal validation beyond form requirements).  
    - Missing or invalid API key results in authorization errors downstream.  
  - Version: 2.2  

#### 2.2 API Request Submission

**Overview:**  
This block sends the user-supplied parameters to KIE.ai’s music generation REST API to initiate a music creation task.

**Nodes Involved:**  
- Send Music Generation Request to KIE.ai API

**Node Details:**  

- **Send Music Generation Request to KIE.ai API**  
  - Type: HTTP Request  
  - Role: Sends POST request to `https://api.kie.ai/api/v1/generate` to start music generation.  
  - Configuration:  
    - Method: POST  
    - Headers:  
      - Content-Type: application/json  
      - Authorization: Bearer token from `api_key` form field  
    - Body JSON includes:  
      - `prompt`: user prompt (music description)  
      - `style`: user specified genre/style  
      - `title`: user specified music title  
      - Fixed parameters: `customMode`=true, `instrumental`=false, `model`="V3_5", `callBackUrl` (placeholder), `negativeTags`=""  
  - Key Expressions: Uses form inputs dynamically via expressions like `{{$json.prompt}}`  
  - Input: JSON from form submission node  
  - Output: API response containing a `taskId` for status polling  
  - Edge Cases:  
    - Authorization failure if API key is invalid  
    - Network errors or timeouts  
    - API returns errors for invalid parameters (e.g., prompt too long)  
  - Version: 4.2  

#### 2.3 Polling and Status Monitoring

**Overview:**  
This block waits a fixed time interval, then polls the API repeatedly to check if music generation has completed, enabling asynchronous processing.

**Nodes Involved:**  
- Wait for Music Processing  
- Poll Music Generation Status  
- Check if Music Generation Complete (If node)

**Node Details:**  

- **Wait for Music Processing**  
  - Type: Wait  
  - Role: Pauses workflow execution for 10 seconds before polling status again  
  - Configuration: Wait duration = 10 seconds  
  - Input: API submission response  
  - Output: Triggers status poll node  
  - Edge Cases:  
    - Excessive waiting if generation takes too long  
    - Potential workflow timeout if API is slow  

- **Poll Music Generation Status**  
  - Type: HTTP Request  
  - Role: Sends GET request to `https://api.kie.ai/api/v1/generate/record-info` with query parameter `taskId` to check generation status  
  - Configuration:  
    - Headers: Content-Type and Authorization (using saved API key from form)  
    - Query param: `taskId` extracted from previous API response (`{{$json.data.taskId}}`)  
  - Input: Triggered after wait node  
  - Output: JSON with status info (`status` field)  
  - Edge Cases:  
    - Task ID invalid or expired  
    - Authorization errors due to missing/invalid API key  
    - API rate limiting or downtime  

- **Check if Music Generation Complete**  
  - Type: If  
  - Role: Evaluates if `data.status` equals "SUCCESS" indicating generation is finished  
  - Configuration: Condition evaluates expression `{{$json.data.status == "SUCCESS"}}` equals `true`  
  - Input: Output of status poll node  
  - Output:  
    - True branch: Proceed to format and display results  
    - False branch: Loop back to wait node for another poll cycle  
  - Edge Cases:  
    - Status stuck in incomplete or error states  
    - Infinite polling loop if no timeout or failure handling  

#### 2.4 Result Retrieval and Formatting

**Overview:**  
This block extracts audio file URLs from the successful API response and formats them for output.

**Nodes Involved:**  
- Format and Display Music Results

**Node Details:**  

- **Format and Display Music Results**  
  - Type: Set  
  - Role: Extracts audio URLs of generated music tracks from nested API response and assigns to output variables  
  - Configuration:  
    - Assigns `audioUrl1` ← `data.response.sunoData[0].audioUrl`  
    - Assigns `audioUrl2` ← `data.response.sunoData[1].audioUrl`  
  - Input: Successful status check node output  
  - Output: JSON with accessible audio URLs for playback or further processing  
  - Edge Cases:  
    - API returning fewer than two audio tracks (may cause index errors)  
    - Missing or malformed response data  

---

### 3. Summary Table

| Node Name                          | Node Type           | Functional Role                           | Input Node(s)                      | Output Node(s)                         | Sticky Note                                                                                                            |
|-----------------------------------|---------------------|-----------------------------------------|----------------------------------|--------------------------------------|------------------------------------------------------------------------------------------------------------------------|
| Sticky Note3                      | Sticky Note         | Overview, prerequisites and setup guide | None                             | None                                 | Contains detailed overview, prerequisites, and setup instructions for the entire workflow                              |
| Sticky Note6                      | Sticky Note         | Step 1: Get API Key instructions         | None                             | None                                 | Explains how to obtain and secure the API key from https://kie.ai/                                                     |
| Sticky Note                       | Sticky Note         | Step 2: Usage process instructions       | None                             | None                                 | Describes step-by-step user workflow from form submission to result retrieval                                          |
| Sticky Note1                     | Sticky Note         | Step 3: Parameter usage guide             | None                             | None                                 | Details usage and constraints of form parameters: prompt, style, title, api_key                                         |
| Submit Music Generation Parameters | Form Trigger        | User input form for music generation     | External form submission         | Send Music Generation Request to KIE.ai API |                                                                                                                        |
| Send Music Generation Request to KIE.ai API | HTTP Request        | Sends POST request to initiate generation | Submit Music Generation Parameters | Wait for Music Processing             |                                                                                                                        |
| Wait for Music Processing         | Wait                | Pauses workflow for polling interval     | Send Music Generation Request to KIE.ai API | Poll Music Generation Status          |                                                                                                                        |
| Poll Music Generation Status      | HTTP Request        | Polls API for generation status           | Wait for Music Processing        | Check if Music Generation Complete    |                                                                                                                        |
| Check if Music Generation Complete| If                  | Checks if generation is complete          | Poll Music Generation Status     | Format and Display Music Results / Wait for Music Processing |                                                                                                                        |
| Format and Display Music Results  | Set                 | Extracts and formats music URLs           | Check if Music Generation Complete (true branch) | None                                 |                                                                                                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Form Trigger node:**  
   - Name: `Submit Music Generation Parameters`  
   - Type: Form Trigger  
   - Configure webhook ID (auto-generated or custom)  
   - Form Title: "AI music generator"  
   - Form Description: "Please fill in the following information to generate your music"  
   - Add form fields (all required):  
     - `prompt` (string, large text)  
     - `style` (string)  
     - `title` (string, max 80 chars)  
     - `api_key` (string, keep secret)  

3. **Add an HTTP Request node:**  
   - Name: `Send Music Generation Request to KIE.ai API`  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.kie.ai/api/v1/generate`  
   - Headers:  
     - Content-Type: application/json  
     - Authorization: Bearer token, set expression: `Bearer {{$json.api_key}}`  
   - Body: raw JSON, set to:  
     ```json
     {
       "prompt": "{{$json.prompt}}",
       "style": "{{$json.style}}",
       "title": "{{$json.title}}",
       "customMode": true,
       "instrumental": false,
       "model": "V3_5",
       "callBackUrl": "https://api.example.com/callback",
       "negativeTags": ""
     }
     ```  
   - Send body as JSON  

4. **Connect the Form Trigger node output to this HTTP Request input.**

5. **Add a Wait node:**  
   - Name: `Wait for Music Processing`  
   - Duration: 10 seconds  

6. **Connect the HTTP Request node output to the Wait node input.**

7. **Add another HTTP Request node:**  
   - Name: `Poll Music Generation Status`  
   - Method: GET  
   - URL: `https://api.kie.ai/api/v1/generate/record-info`  
   - Query Parameters:  
     - `taskId` set as expression: `{{$json.data.taskId}}` (from previous API response)  
   - Headers:  
     - Content-Type: application/json  
     - Authorization: Bearer token from the API key originally submitted:  
       Use expression: `Bearer {{$node["Submit Music Generation Parameters"].json["api_key"]}}`  

8. **Connect the Wait node output to this Poll node input.**

9. **Add an If node:**  
   - Name: `Check if Music Generation Complete`  
   - Condition: Check if expression `{{$json.data.status == "SUCCESS"}}` is true (string equals true)  

10. **Connect Poll node output to the If node input.**

11. **Add a Set node:**  
    - Name: `Format and Display Music Results`  
    - Set fields:  
      - `audioUrl1` → expression: `{{$json.data.response.sunoData[0].audioUrl}}`  
      - `audioUrl2` → expression: `{{$json.data.response.sunoData[1].audioUrl}}`  

12. **Connect the If node true output to this Set node.**

13. **Connect the If node false output back to the Wait node** to continue polling.

14. **Save and activate the workflow.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               | Context or Link                       |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------|
| Workflow allows iterative polling every 10 seconds to check music generation status, which may cause longer waits for complex or slow generation tasks. Consider adding a maximum retry or timeout logic if needed.                                                                                                                                                                                                                                                | Workflow design consideration       |
| For improved prompt results, include detailed musical moods, instrument types, and rhythm descriptions. Examples: "A peaceful piano meditation track with gentle waves in the background."                                                                                                                                                                                                                                                                                                                     | Sticky Note1 guidance                |
| Obtain your API key safely at [KIE.ai](https://kie.ai). Never expose your API key publicly or share it in unsecured environments.                                                                                                                                                                                                                                                                                                                                                                         | Sticky Note6 instructions            |
| The `callBackUrl` in the API request is a placeholder and not used by this workflow; consider removing or customizing for webhook callback support if KIE.ai supports it.                                                                                                                                                                                                                                                                                                                                  | API integration note                 |
| The workflow is designed for n8n version supporting Form Trigger v2.2 and HTTP Request v4.2 nodes to ensure compatibility with expression usage and API features.                                                                                                                                                                                                                                                                                                                                        | Version compatibility                |
| This workflow template is ideal for developers seeking to automate AI music creation integrated with forms and REST APIs without custom coding.                                                                                                                                                                                                                                                                                                                                                            | Overall purpose                     |

---

**Disclaimer:**  
The provided text is exclusively derived from an automated n8n workflow and complies fully with applicable content policies. It contains no illegal or offensive material. All processed data is legal and public.