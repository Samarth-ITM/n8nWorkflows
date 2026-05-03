Generate AI Videos from Text Prompts with KIE.AI Veo3

https://n8nworkflows.xyz/workflows/generate-ai-videos-from-text-prompts-with-kie-ai-veo3-6047


# Generate AI Videos from Text Prompts with KIE.AI Veo3

### 1. Workflow Overview

This n8n workflow, titled **"Generate AI Videos from Text Prompts with KIE.AI Veo3"**, automates the text-to-video generation process using the KIE.AI Veo3 API. It is designed for users who want to create AI-generated videos by submitting descriptive text prompts via a web form. The workflow handles request submission, asynchronous video generation monitoring, and result retrieval, making it suitable for content creators, marketers, and developers exploring AI-driven video production.

The workflow is structured into the following logical blocks:

- **1.1 User Input Reception**: Presents a form to the user to submit the video prompt and API key.
- **1.2 Video Generation Request Submission**: Sends the user input to the KIE.AI API to start video generation.
- **1.3 Video Processing Monitoring**: Waits and polls the API to check if the video generation is complete.
- **1.4 Result Retrieval and Presentation**: Once generation is done, formats and outputs the URLs of the generated video files.
- **1.5 Documentation and User Guidance**: Sticky notes provide instructions, prerequisites, and usage steps to facilitate correct operation.

---

### 2. Block-by-Block Analysis

#### 1.1 User Input Reception

- **Overview:**  
  This block collects the text prompt describing the desired video and the user's API key through a web form, serving as the entry point of the workflow.

- **Nodes Involved:**  
  - Submit Text Prompt for Video Generation (Form Trigger)

- **Node Details:**

  - **Submit Text Prompt for Video Generation**
    - Type: Form Trigger  
      Enables a user-facing form that triggers the workflow upon submission.
    - Configuration:
      - Form Title: "AI video generator"
      - Form Description: "Please fill in the following information to generate your video"
      - Fields:
        - `prompt`: User’s description of the video content.
        - `api_key`: KIE.AI API key for authentication.
    - Expressions / Variables:
      - Values submitted by the user are accessible as JSON properties (`$json.prompt`, `$json.api_key`).
    - Input / Output:
      - No input (trigger node).
      - Output passes form data to the next node.
    - Edge Cases / Failures:
      - Missing or invalid API key leads to authorization failures downstream.
      - Empty or poorly formatted prompt may cause API errors or low-quality output.
    - Version-Specific Requirements: Version 2.2 or higher supports enhanced form features.

---

#### 1.2 Video Generation Request Submission

- **Overview:**  
  Sends the submitted prompt and API key to the KIE.AI Veo3 API endpoint to initiate video generation.

- **Nodes Involved:**  
  - Send Video Generation Request to KIE.AI API

- **Node Details:**

  - **Send Video Generation Request to KIE.AI API**
    - Type: HTTP Request  
      Sends a POST request with JSON payload to the video generation API.
    - Configuration:
      - URL: `https://api.kie.ai/api/v1/veo/generate`
      - Method: POST
      - Headers:
        - `Content-Type`: application/json
        - `Authorization`: Bearer token from user input (`Bearer {{$json.api_key}}`)
      - Body (JSON):
        ```json
        {
          "prompt": "{{$json.prompt}}",
          "model": "veo3",
          "watermark": "",
          "callBackUrl": "https://api.example.com/callback",
          "aspectRatio": "16:9",
          "seeds": 12345
        }
        ```
      - Sends body and headers as JSON.
    - Expressions / Variables:
      - `prompt` and `api_key` dynamically inserted from form data.
    - Input / Output:
      - Input: JSON from form trigger.
      - Output: Response JSON includes a `taskId` for monitoring.
    - Edge Cases / Failures:
      - Authentication failure if API key is invalid.
      - API downtime or network errors.
      - Malformed prompt or unsupported parameters causing rejection.
    - Notes:
      - `callBackUrl` is set but appears as a placeholder (`https://api.example.com/callback`) and is not actively used in this workflow.

---

#### 1.3 Video Processing Monitoring

- **Overview:**  
  Implements a polling loop that waits and queries the API every 10 seconds to track the video generation status until completion.

- **Nodes Involved:**  
  - Wait for Video Processing Completion  
  - Obtain the generated status  
  - Check if Video Generation is Complete (IF node)

- **Node Details:**

  - **Wait for Video Processing Completion**
    - Type: Wait  
      Pauses workflow execution for 10 seconds between status checks.
    - Configuration:
      - Amount: 10 seconds
    - Input / Output:
      - Input: Response from previous node.
      - Output: Triggers the HTTP request to check status.
    - Edge Cases:
      - Excessive waiting if video generation takes long, potentially causing timeout in some execution environments.

  - **Obtain the generated status**
    - Type: HTTP Request  
      Polls the API endpoint to get current task status.
    - Configuration:
      - URL: `https://api.kie.ai/api/v1/veo/record-info`
      - Method: GET (default)
      - Query parameter: `taskId` from previous response (`{{$json.data.taskId}}`)
      - Headers:
        - `Content-Type`: application/json
        - `Authorization`: Bearer token from original form input (`{{$node["Submit Text Prompt for Video Generation"].json["api_key"]}}`)
    - Input / Output:
      - Input: Wait node output.
      - Output: JSON with status and success flag.
    - Edge Cases:
      - Invalid or expired `taskId` causes error.
      - Authentication failure if API key changed or revoked.
      - API rate limits could interrupt polling.

  - **Check if Video Generation is Complete**
    - Type: IF  
      Determines if the video generation succeeded to proceed or continue waiting.
    - Configuration:
      - Condition: Checks if `$json.data.successFlag == 1` equals true.
    - Input / Output:
      - Input: Status response.
      - Output:
        - True: Proceed to format results.
        - False: Loop back to wait node to retry after 10 seconds.
    - Edge Cases:
      - If the flag never becomes `1`, workflow loops indefinitely unless externally stopped.
      - API changes in response schema could break condition.

---

#### 1.4 Result Retrieval and Presentation

- **Overview:**  
  Once the video generation is complete, extracts and formats the URLs of the original and processed video files for user access.

- **Nodes Involved:**  
  - Format and Display Video Results

- **Node Details:**

  - **Format and Display Video Results**
    - Type: Set  
      Creates output fields containing video URLs for display or further use.
    - Configuration:
      - Assigns:
        - `originUrls` = `{{$json.data.response.originUrls}}`
        - `resultUrls` = `{{$json.data.response.resultUrls}}`
      - Clears any unused fields.
    - Input / Output:
      - Input: Successful generation status JSON.
      - Output: JSON containing URLs for download or embedding.
    - Edge Cases:
      - Missing URLs if generation failed silently.
      - Format incompatible with UI expecting other data.

---

#### 1.5 Documentation and User Guidance

- **Overview:**  
  Several sticky notes provide comprehensive instructions, prerequisites, and usage steps directly within the workflow canvas for user reference.

- **Nodes Involved:**  
  - Sticky Note6  
  - Sticky Note  
  - Sticky Note3  
  - Sticky Note1

- **Node Details:**

  - **Sticky Notes**
    - Type: Sticky Note  
      Non-executable nodes containing markdown content.
    - Content highlights:
      - **Sticky Note6:** Instructions to obtain API key from [kie.ai](https://kie.ai/) with emphasis on security.
      - **Sticky Note:** Detailed usage process from starting workflow to retrieving results.
      - **Sticky Note3:** Overview of workflow purpose, prerequisites, setup instructions, and customization tips.
      - **Sticky Note1:** Explanation of form parameters (`prompt` and `api_key`) with tips for effective prompts.
    - Purpose:
      - Helps users understand setup and operation without external documentation.
    - Position:
      - Strategically placed near relevant nodes for contextual clarity.

---

### 3. Summary Table

| Node Name                           | Node Type           | Functional Role                           | Input Node(s)                               | Output Node(s)                            | Sticky Note                                                                                                   |
|-----------------------------------|---------------------|-----------------------------------------|---------------------------------------------|-------------------------------------------|--------------------------------------------------------------------------------------------------------------|
| Submit Text Prompt for Video Generation | Form Trigger        | Collect user video prompt and API key   | None                                        | Send Video Generation Request to KIE.AI API | See Sticky Note1 for form parameter details. See Sticky Note6 for API key acquisition instructions.          |
| Send Video Generation Request to KIE.AI API | HTTP Request       | Initiate video generation via API       | Submit Text Prompt for Video Generation      | Wait for Video Processing Completion       | See Sticky Note for overall usage process.                                                                    |
| Wait for Video Processing Completion | Wait                | Pause before polling for completion     | Send Video Generation Request to KIE.AI API | Obtain the generated status                 | See Sticky Note for polling instructions and waiting details.                                                |
| Obtain the generated status        | HTTP Request        | Poll API for current video generation status | Wait for Video Processing Completion         | Check if Video Generation is Complete       | See Sticky Note for usage process.                                                                            |
| Check if Video Generation is Complete | IF                  | Check if generation succeeded           | Obtain the generated status                   | Format and Display Video Results (true) / Wait for Video Processing Completion (false) | See Sticky Note for usage process.                                                                            |
| Format and Display Video Results   | Set                 | Format and output video URLs             | Check if Video Generation is Complete (true) | None                                      | See Sticky Note for final output handling.                                                                    |
| Sticky Note6                      | Sticky Note          | API key acquisition instructions        | None                                        | None                                      | Contains [kie.ai](https://kie.ai/) registration and API key security instructions.                            |
| Sticky Note                      | Sticky Note          | Usage process and workflow steps         | None                                        | None                                      | Provides stepwise usage instructions from start to result retrieval.                                         |
| Sticky Note3                     | Sticky Note          | Workflow overview and setup instructions | None                                        | None                                      | Detailed workflow overview, prerequisites, setup, and customization tips.                                    |
| Sticky Note1                     | Sticky Note          | Form parameter explanations               | None                                        | None                                      | Explains form fields `prompt` and `api_key` with usage tips.                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node:**  
   - Name: `Submit Text Prompt for Video Generation`  
   - Type: Form Trigger (v2.2 or later)  
   - Configure form:  
     - Title: "AI video generator"  
     - Description: "Please fill in the following information to generate your video"  
     - Fields:  
       - `prompt` (text input for video description)  
       - `api_key` (text input for KIE.AI API key)  
   - This node will serve as the workflow starting point.

2. **Add HTTP Request Node to Submit Generation Request:**  
   - Name: `Send Video Generation Request to KIE.AI API`  
   - Type: HTTP Request (v4.2)  
   - Method: POST  
   - URL: `https://api.kie.ai/api/v1/veo/generate`  
   - Headers:  
     - `Content-Type`: `application/json`  
     - `Authorization`: `Bearer {{$json.api_key}}` (expression from form input)  
   - Body (JSON):  
     ```json
     {
       "prompt": "{{$json.prompt}}",
       "model": "veo3",
       "watermark": "",
       "callBackUrl": "https://api.example.com/callback",
       "aspectRatio": "16:9",
       "seeds": 12345
     }
     ```
   - Connect output of Form Trigger node to this HTTP Request node.

3. **Insert Wait Node:**  
   - Name: `Wait for Video Processing Completion`  
   - Type: Wait (v1.1)  
   - Configure to wait 10 seconds (`amount`: 10)  
   - Connect output of HTTP Request node (send generation request) to this Wait node.

4. **Add HTTP Request Node to Poll Generation Status:**  
   - Name: `Obtain the generated status`  
   - Type: HTTP Request (v4.2)  
   - Method: GET (default)  
   - URL: `https://api.kie.ai/api/v1/veo/record-info`  
   - Query Parameter:  
     - `taskId`: `={{$json.data.taskId}}` (extracted from prior generation request response)  
   - Headers:  
     - `Content-Type`: `application/json`  
     - `Authorization`: `Bearer {{$node["Submit Text Prompt for Video Generation"].json["api_key"]}}` (original user API key)  
   - Connect output of Wait node to this node.

5. **Add IF Node to Check Completion:**  
   - Name: `Check if Video Generation is Complete`  
   - Type: IF (v2.2)  
   - Condition:  
     - Expression: `$json.data.successFlag == 1` must be true (string equals "true")  
   - Connect output of status HTTP Request node to this IF node.

6. **On IF True Branch, Add Set Node for Results:**  
   - Name: `Format and Display Video Results`  
   - Type: Set (v3.4)  
   - Configure to assign:  
     - `originUrls` = `{{$json.data.response.originUrls}}`  
     - `resultUrls` = `{{$json.data.response.resultUrls}}`  
   - Connect IF node’s true output to this Set node.

7. **On IF False Branch, Loop Back:**  
   - Connect IF node’s false output back to the Wait node to repeat polling every 10 seconds.

8. **Add Sticky Notes for Documentation:**  
   - Add four Sticky Notes around the workflow to provide user instructions:  
     - API key acquisition and security (with [kie.ai](https://kie.ai) link)  
     - Overall usage steps  
     - Workflow overview, prerequisites, and setup guidelines  
     - Form field explanations and prompt tips

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                             | Context or Link                          |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------|
| Obtain your API key by creating an account at [https://kie.ai/](https://kie.ai/). Keep your API key secure and do not disclose it to others.                            | Sticky Note6                           |
| To use the workflow: start it, fill the form with a descriptive prompt and your API key, submit, then wait as the system polls the API every 10 seconds for completion. | Sticky Note                           |
| The workflow uses KIE.AI Veo3 model for AI video generation from text prompts, requiring n8n with HTTP Request and form trigger capabilities.                            | Sticky Note3                          |
| For best results, provide detailed prompts including scene actions, style, and visual elements. Examples: "A serene mountain landscape at sunset with birds flying."    | Sticky Note1                          |

---

**Disclaimer:** The provided content originates exclusively from an automated workflow created with n8n, a workflow automation tool. It complies strictly with content policies and contains no illegal, offensive, or protected elements. All data processed is legal and publicly accessible.