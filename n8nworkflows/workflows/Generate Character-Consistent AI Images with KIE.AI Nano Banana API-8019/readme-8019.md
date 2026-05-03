Generate Character-Consistent AI Images with KIE.AI Nano Banana API

https://n8nworkflows.xyz/workflows/generate-character-consistent-ai-images-with-kie-ai-nano-banana-api-8019


# Generate Character-Consistent AI Images with KIE.AI Nano Banana API

---
### 1. Workflow Overview

This n8n workflow, titled **"Generate Character-Consistent AI Images with KIE.AI Nano Banana API"**, is designed to facilitate the generation of AI-driven images that maintain character consistency using KIE.AI’s Nano Banana API. It supports two generation modes: **text-to-image** and **image-to-image**, providing an easy-to-use form interface for users to submit prompts, reference images, and their API key, and then automatically processes the request until final images are ready.

The workflow is logically divided into the following blocks:

- **1.1 User Input & Form Submission:** Presents a web form for users to enter generation parameters (prompt, optional reference images, model type, API key).
- **1.2 API Request to KIE.AI Nano Banana:** Sends the image generation request to the KIE.AI API using the submitted data.
- **1.3 Polling for Generation Status:** Periodically checks the status of the image generation task until completion.
- **1.4 Result Handling & Display:** Once generation is complete, formats and outputs the resulting images.
- **1.5 Documentation & User Guidance:** Multiple sticky notes provide step-by-step instructions, usage tips, and parameter explanations for users.

---

### 2. Block-by-Block Analysis

#### 2.1 User Input & Form Submission

- **Overview:**  
  This block collects user input via a web form. The form requests the AI prompt, optional reference image URLs, model selection (text-to-image or image-to-image), and the user's KIE.AI API key. It acts as the workflow’s trigger point.

- **Nodes Involved:**  
  - `Submit Text Prompt for image Generation`

- **Node Details:**  
  - **Submit Text Prompt for image Generation**  
    - *Type:* Form Trigger node  
    - *Role:* Captures form data submitted by users to start the image generation process.  
    - *Configuration:*  
      - Form title: "AI image generator"  
      - Form fields: prompt (text), img_url (optional, comma-separated URLs), model (choice between `google/nano-banana` and `google/nano-banana-edit`), api_key (user’s API key)  
      - Webhook ID assigned for form submissions  
    - *Expressions:* Uses `$json` to access submitted form data for downstream nodes.  
    - *Connections:* Outputs to the HTTP Request node that sends generation requests.  
    - *Edge Cases:*  
      - Missing or invalid API key will cause authorization failure downstream.  
      - Improperly formatted `img_url` (e.g., malformed URLs) may cause request errors.  
      - Model selection inconsistent with presence of `img_url` (e.g., image URLs supplied but text-to-image selected) could yield unexpected behavior.

#### 2.2 API Request to KIE.AI Nano Banana

- **Overview:**  
  This block constructs and sends the image generation task request to the KIE.AI Nano Banana API endpoint, using user inputs from the form.

- **Nodes Involved:**  
  - `Send image Generation Request to KIE.AI API`

- **Node Details:**  
  - **Send image Generation Request to KIE.AI API**  
    - *Type:* HTTP Request node  
    - *Role:* Posts a JSON payload to the KIE.AI API to initiate image generation.  
    - *Configuration:*  
      - Method: POST  
      - URL: `https://api.kie.ai/api/v1/playground/createTask`  
      - Headers: `Content-Type: application/json`, `Authorization: Bearer <api_key>` (from form input)  
      - Body (JSON):  
        ```json
        {
          "model": "<model>",
          "callBackUrl": "https://your-domain.com/api/callback",
          "input": {
            "prompt": "<prompt>",
            "image_urls": [ "<list of trimmed image URLs or empty>" ]
          }
        }
        ```  
      - The `image_urls` array is built dynamically by splitting the `img_url` string on commas and trimming whitespace; empty if no URLs provided.  
    - *Expressions:* Uses `$json` variables from the form submission node.  
    - *Connections:* Outputs to the Wait node for processing completion.  
    - *Edge Cases:*  
      - Network errors or API downtime could cause POST failure.  
      - Invalid API key or insufficient credits will cause auth or quota errors.  
      - Invalid model name or malformed prompt payload could cause API rejection.  
      - If `img_url` is improperly formatted, API may reject or ignore image URLs.

#### 2.3 Polling for Generation Status

- **Overview:**  
  This block implements a polling mechanism to repeatedly check whether the image generation task has completed, querying the KIE.AI API every 5 seconds until success.

- **Nodes Involved:**  
  - `Wait for image Processing Completion`  
  - `Obtain the generated status`  
  - `Check if image Generation is Complete`

- **Node Details:**  
  - **Wait for image Processing Completion**  
    - *Type:* Wait node  
    - *Role:* Introduces delay between status checks (acts as a 5-second timer).  
    - *Configuration:* Default wait time (not explicitly stated, but implied by sticky notes as 5 seconds).  
    - *Connections:* After wait, triggers status check HTTP request node.  
    - *Edge Cases:*  
      - Long generation times may cause extended polling; workflow must handle timeouts gracefully.  
  - **Obtain the generated status**  
    - *Type:* HTTP Request node  
    - *Role:* GETs the status of the submitted task from KIE.AI API.  
    - *Configuration:*  
      - URL: `https://api.kie.ai/api/v1/playground/recordInfo`  
      - Query parameter: `taskId` obtained dynamically from previous API response (`$json.data.taskId`)  
      - Headers: `Authorization` with bearer token from original API key  
    - *Connections:* Outputs to the IF node that evaluates completion status.  
    - *Edge Cases:*  
      - API errors or network issues may cause request failure.  
      - Missing or expired task ID would cause invalid responses.  
  - **Check if image Generation is Complete**  
    - *Type:* IF node  
    - *Role:* Evaluates if the `state` field in the API response equals `"success"`.  
    - *Configuration:* Condition: `{{$json.data.state == 'success'}}` equals true.  
    - *Connections:*  
      - If true: proceeds to format and display results.  
      - If false: loops back to wait node for another polling iteration.  
    - *Edge Cases:*  
      - API may return other states like "pending", "failed", or "error" which are implicitly handled by looping or halting.  
      - Infinite loops possible if failure states not explicitly handled—this workflow relies on manual interruption or timeouts.

#### 2.4 Result Handling & Display

- **Overview:**  
  Upon successful generation, this block extracts the resulting image data and prepares it for display or further use.

- **Nodes Involved:**  
  - `Format and Display image Results`

- **Node Details:**  
  - **Format and Display image Results**  
    - *Type:* Set node  
    - *Role:* Assigns the generated image data from API response JSON to a new field named `img` for downstream consumption or display.  
    - *Configuration:*  
      - Sets `img` string field to the contents of `$json.data.resultJson` received from the generation status response.  
    - *Connections:* Terminal node in the current workflow.  
    - *Edge Cases:*  
      - If `resultJson` is empty or malformed, the output will be invalid or empty.  
      - No explicit error handling if final image data is missing; could be improved.

#### 2.5 Documentation & User Guidance (Sticky Notes)

- **Overview:**  
  Multiple sticky notes provide stepwise instructions, usage tips, parameter explanations, and overall workflow context to assist users in understanding and operating the workflow.

- **Nodes Involved:**  
  - `Sticky Note3` (comprehensive overview and usage guide)  
  - `Sticky Note6` (API key acquisition instructions)  
  - `Sticky Note` (usage process steps)  
  - `Sticky Note1` (form parameter explanations)

- **Node Details:**  
  - Sticky notes contain helpful markdown-formatted content, including links to KIE.AI signup pages and best practices for prompt engineering and parameter usage.  
  - Positioned around the workspace for visibility during workflow editing or execution.  
  - No direct data flow connections; purely informational.

---

### 3. Summary Table

| Node Name                             | Node Type               | Functional Role                          | Input Node(s)                      | Output Node(s)                          | Sticky Note                                                                                     |
|-------------------------------------|-------------------------|----------------------------------------|----------------------------------|---------------------------------------|------------------------------------------------------------------------------------------------|
| Submit Text Prompt for image Generation | Form Trigger            | Captures user inputs via web form      | (Trigger)                        | Send image Generation Request to KIE.AI API | See Sticky Note1 for form parameters explanation                                             |
| Send image Generation Request to KIE.AI API | HTTP Request            | Sends image generation task to API     | Submit Text Prompt for image Generation | Wait for image Processing Completion    |                                                                                                |
| Wait for image Processing Completion | Wait                    | Delays between polling attempts         | Send image Generation Request to KIE.AI API | Obtain the generated status             | See Sticky Note (usage process) for polling steps                                             |
| Obtain the generated status          | HTTP Request            | Requests current status of generation   | Wait for image Processing Completion | Check if image Generation is Complete   |                                                                                                |
| Check if image Generation is Complete | IF                      | Checks if generation is complete        | Obtain the generated status       | Format and Display image Results / Wait for image Processing Completion |                                                                                                |
| Format and Display image Results     | Set                     | Prepares final image data for output    | Check if image Generation is Complete (true path) | (Terminal)                             |                                                                                                |
| Sticky Note3                        | Sticky Note             | Provides detailed overview and usage guide | (None)                          | (None)                                | See full overview and workflow context                                                        |
| Sticky Note6                        | Sticky Note             | Instructions for obtaining API Key      | (None)                          | (None)                                | See instructions for API key acquisition                                                      |
| Sticky Note                         | Sticky Note             | Stepwise usage process instructions     | (None)                          | (None)                                | Step-by-step instructions for workflow execution                                              |
| Sticky Note1                       | Sticky Note             | Explains form parameters                | (None)                          | (None)                                | Detailed form parameter descriptions                                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Form Trigger Node**  
   - Name: `Submit Text Prompt for image Generation`  
   - Type: Form Trigger  
   - Configure webhook ID (auto-generated or custom)  
   - Form Title: "AI image generator"  
   - Form Fields:  
     - `prompt` (Text, required)  
     - `img_url` (Text, optional; comma-separated URLs)  
     - `model` (Dropdown or Text, required; options: `google/nano-banana`, `google/nano-banana-edit`)  
     - `api_key` (Text, required)  
   - Description: "Please fill in the following information to generate your video"  

2. **Create HTTP Request Node to Send Generation Task**  
   - Name: `Send image Generation Request to KIE.AI API`  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.kie.ai/api/v1/playground/createTask`  
   - Headers:  
     - `Content-Type`: `application/json`  
     - `Authorization`: `Bearer {{$json.api_key}}` (expression referencing form input)  
   - Body (JSON):  
     ```json
     {
       "model": "{{$json.model}}",
       "callBackUrl": "https://your-domain.com/api/callback",
       "input": {
         "prompt": "{{$json.prompt}}",
         "image_urls": [{{$json.img_url ? $json.img_url.split(',').map(url => '\"' + url.trim() + '\"').join(',') : ''}}]
       }
     }
     ```  
   - Set "Send Body" and "Send Headers" to true.  
   - Connect output of form trigger node to this node.

3. **Add Wait Node for Polling Delay**  
   - Name: `Wait for image Processing Completion`  
   - Type: Wait  
   - Configure delay duration: 5 seconds (or default if not explicitly set)  
   - Connect output of the HTTP request node to this node.

4. **Create HTTP Request Node to Obtain Generation Status**  
   - Name: `Obtain the generated status`  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://api.kie.ai/api/v1/playground/recordInfo`  
   - Query Parameters:  
     - `taskId`: `={{$json.data.taskId}}` (pulled from previous task creation response)  
   - Headers:  
     - `Content-Type`: `application/json`  
     - `Authorization`: `Bearer {{$node["Submit Text Prompt for image Generation"].json["api_key"]}}`  
   - Connect output of Wait node to this node.

5. **Add IF Node to Check Completion**  
   - Name: `Check if image Generation is Complete`  
   - Type: IF  
   - Condition: Check if `{{$json.data.state}}` equals `"success"`  
   - If true: connect to the formatting node  
   - If false: loop back to the Wait node to poll again.

6. **Create Set Node to Format and Display Results**  
   - Name: `Format and Display image Results`  
   - Type: Set  
   - Add field:  
     - Name: `img` (string)  
     - Value: `{{$json.data.resultJson}}` (contains generated image data)  
   - Connect "true" output from IF node here.

7. **Add Sticky Notes for Documentation (Optional but Recommended)**  
   - Create sticky notes with the following content for user guidance and maintenance:  
     - API key acquisition instructions with signup link to https://kie.ai/  
     - Usage process steps for executing the workflow and submitting the form  
     - Detailed form parameters explanation including model options and examples  
     - Full workflow overview and benefits  

8. **Configure Credentials**  
   - No explicit credential node needed; API key is supplied dynamically via form input.  
   - Ensure users keep their API key secure and do not expose it publicly.

9. **Activate and Test the Workflow**  
   - Save and activate the workflow.  
   - Execute workflow and access the form URL generated by the Form Trigger node.  
   - Submit valid inputs and observe workflow progress and final image results.

---

### 5. General Notes & Resources

| Note Content                                                                                                                  | Context or Link                                 |
|------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------|
| Create an account and obtain your KIE.AI Nano Banana API key at [https://kie.ai/](https://kie.ai/)                             | API Key acquisition instructions                |
| Workflow supports both text-to-image (`google/nano-banana`) and image-to-image (`google/nano-banana-edit`) generation modes    | Model options description                        |
| Use detailed prompts with style, composition, lighting, and mood for best results                                             | Prompt engineering tips                          |
| The workflow polls the API every 5 seconds until generation is complete                                                       | Polling mechanism explanation                    |
| Costs: 20 free generations, then $0.02 per request                                                                           | Pricing information                              |
| Ensure that image URLs for image-to-image mode are valid and separated by commas (max 5 images)                               | Usage tips for `img_url` parameter               |
| Typical use cases: character design, marketing materials, social media content, product mockups                              | Use case ideas                                   |
| Troubleshooting tips: verify API key validity, check prompt content and length                                                | Common issues and fixes                           |

---

**Disclaimer:** The provided text originates exclusively from an automated n8n workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.