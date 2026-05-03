Generate AI Images with Text Prompts using BananaAPI and Nano Banana Engine

https://n8nworkflows.xyz/workflows/generate-ai-images-with-text-prompts-using-bananaapi-and-nano-banana-engine-8375


# Generate AI Images with Text Prompts using BananaAPI and Nano Banana Engine

### 1. Workflow Overview

This workflow enables users to generate AI images based on text prompts by integrating the BananaAPI.com image generation service and its Nano Banana Engine. It is designed to accept user inputs (prompts, optional image references, and configuration) through a web form, submit these to BananaAPI, poll the task status until completion, and finally return the generated image URL along with task metadata.

**Target Use Cases:**  
- Content creators and marketers seeking affordable AI-generated images  
- Developers automating image generation in applications  
- Experimentation with AI image prompts without subscription constraints

**Logical Blocks:**

- **1.1 Input Reception:** User submits prompt, API token, and optional parameters via a form trigger.  
- **1.2 API Submission:** Sends prompt and options to BananaAPI via HTTP POST, retrieving a task ID.  
- **1.3 Status Polling Loop:** Waits and periodically checks task status until the image generation completes.  
- **1.4 Output Preparation:** Once completed, formats and returns the image URL and task details.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  Captures user inputs via a web-accessible form to provide the necessary data for image generation.

- **Nodes Involved:**  
  - Form — Get Prompt

- **Node Details:**

  - **Form — Get Prompt**  
    - *Type & Role:* Form Trigger node; entry point capturing user data via a web form.  
    - *Configuration:*  
      - Form titled "Banana API — Image Generator".  
      - Fields:  
        - `api_token` (required string, with placeholder "Bearer xxxxxxx...")  
        - `prompt` (required textarea, example: "a neon cyberpunk cat, detailed, 4k")  
        - `Output Format [optional]` (dropdown: png, jpeg)  
        - `Image Size [optional]` (dropdown: 16:9, 9:16, 1:1, 3:4, 4:3)  
        - `image_url_1` to `image_url_5` (optional URLs for reference images)  
      - Form description instructs users to enter prompt and optionally images.  
    - *Inputs:* None (trigger node)  
    - *Outputs:* Data object containing all form fields.  
    - *Version-specific:* Uses webhook-based form trigger (v2.2).  
    - *Potential Failures:* Missing required fields; invalid URLs in optional fields; user cancellation.  
    - *Notes:* Uses direct user input, so data validation on client-side recommended.

#### 2.2 API Submission

- **Overview:**  
  Sends user inputs to BananaAPI to request image generation and obtains a task ID to track progress.

- **Nodes Involved:**  
  - Submit — Banana API

- **Node Details:**

  - **Submit — Banana API**  
    - *Type & Role:* HTTP Request node performing POST to BananaAPI’s generation endpoint.  
    - *Configuration:*  
      - URL: `https://bananaapi.com/api/n8n/generate/`  
      - Method: POST  
      - Headers:  
        - `Content-Type: application/json`  
        - `Authorization: Bearer {{ $json.api_token }}` (dynamic from form input)  
      - Body (JSON):  
        ```json
        {
          "input": {
            "prompt": "{{ $json.prompt }}",
            "image_size": "{{ $json['Image Size [optional]'] }}",
            "output_format": "{{ $json['Output Format [optional]'] }}",
            "image_urls": [
              "{{ $json['image_url_1 [optional]'] }}",
              "{{ $json['image_url_2 [optional]'] }}",
              "{{ $json['image_url_3 [optional]'] }}",
              "{{ $json['image_url_4 [optional]'] }}",
              "{{ $json['image_url_5 [optional]'] }}"
            ]
          }
        }
        ```  
      - Sends body and headers as JSON.  
    - *Inputs:* Receives JSON from Form node with prompt and tokens.  
    - *Outputs:* Response containing a `taskId` for tracking image generation.  
    - *Version-specific:* HTTP Request node version 4.2 supports templated JSON body.  
    - *Potential Failures:*  
      - 401/403 Unauthorized if API token invalid or missing  
      - Malformed JSON body or invalid URL fields causing 400 errors  
      - Network timeouts or BananaAPI service downtime  
    - *Notes:* API key must be kept confidential; recommended to store as n8n Credential for production.

#### 2.3 Status Polling Loop

- **Overview:**  
  Repeatedly checks the status of the image generation task until it is marked as completed.

- **Nodes Involved:**  
  - Wait 5s  
  - Check Status  
  - If  
  - Wait 5s (loop)

- **Node Details:**

  - **Wait 5s**  
    - *Type & Role:* Wait node to delay workflow for 5 seconds before status check.  
    - *Configuration:* Default wait time, no parameters needed.  
    - *Inputs:* From Submit — Banana API node.  
    - *Outputs:* To Check Status node.  
    - *Potential Failures:* None significant; minor delay in processing.

  - **Check Status**  
    - *Type & Role:* HTTP Request node performing GET request to retrieve task status.  
    - *Configuration:*  
      - URL templated as:  
        `https://bananaapi.com/api/n8n/image-status/{{$('Submit — Banana API').item.json.taskId}}`  
      - Method: GET (default)  
    - *Inputs:* From Wait 5s or Wait 5s (loop) nodes.  
    - *Outputs:* Response JSON with status and possibly image URL.  
    - *Potential Failures:*  
      - 404 or invalid taskId  
      - Network issues  
      - Delayed task completion causing repeated polling

  - **If**  
    - *Type & Role:* Conditional node that checks if the task status equals "completed".  
    - *Configuration:*  
      - Condition:  
        Left: `{{$('Check Status').item.json.status}}`  
        Operator: equals  
        Right: "completed"  
    - *Inputs:* From Check Status node.  
    - *Outputs:*  
      - True branch: task completed → proceeds to output formatting  
      - False branch: task not completed → loops back to Wait 5s (loop)  
    - *Potential Failures:* Expression evaluation errors if `status` field missing.

  - **Wait 5s (loop)**  
    - *Type & Role:* Wait node to delay 5 seconds before next status check iteration.  
    - *Configuration:* Default wait time.  
    - *Inputs:* False branch from If node.  
    - *Outputs:* Loops back to Check Status node.

#### 2.4 Output Preparation

- **Overview:**  
  Formats and outputs the final image URL, task ID, and status once the generation completes.

- **Nodes Involved:**  
  - Return Links

- **Node Details:**

  - **Return Links**  
    - *Type & Role:* Set node to prepare output data for downstream consumption or API response.  
    - *Configuration:*  
      - Assigns:  
        - `image_url` = value from current JSON `imageUrl` field (from Check Status response)  
        - `task_id` = `taskId` from Submit — Banana API node JSON  
        - `status` = current `status` field from Check Status response  
    - *Inputs:* True branch from If node.  
    - *Outputs:* Set formatted JSON object for output.  
    - *Potential Failures:* Missing `imageUrl` if API response incomplete.

---

### 3. Summary Table

| Node Name           | Node Type           | Functional Role                  | Input Node(s)        | Output Node(s)       | Sticky Note                                                                                                           |
|---------------------|---------------------|--------------------------------|----------------------|----------------------|-----------------------------------------------------------------------------------------------------------------------|
| Form — Get Prompt    | Form Trigger        | Capture user inputs via form    | None                 | Submit — Banana API   | ## Step 1 — User Form Input: api_token, prompt, output format, image size, optional image URLs                         |
| Submit — Banana API  | HTTP Request        | Submit prompt to BananaAPI      | Form — Get Prompt    | Wait 5s              | ## Step 2 — Submit to Banana API: POST with prompt, image size, output format, image URLs, returns taskId             |
| Wait 5s             | Wait                | Delay before status check       | Submit — Banana API  | Check Status         | ## Step 3 — Processing: wait 5s, then check status                                                                    |
| Check Status        | HTTP Request        | Query task status from BananaAPI| Wait 5s, Wait 5s(loop) | If                   | ## API Reference: GET /image-status/{taskId} with Authorization header                                               |
| If                  | If                  | Check if task status is complete| Check Status         | Return Links (true), Wait 5s (loop) (false) | ## Troubleshooting: 401/403 errors, invalid JSON, slow performance                                                   |
| Wait 5s (loop)       | Wait                | Delay for polling loop          | If (false branch)    | Check Status         | ## Step 3 — Processing: loop waits 5s and rechecks status                                                             |
| Return Links         | Set                 | Format output data              | If (true branch)     | None                 | ## Step 4 — Return Output: final image URL, task ID, status; tip to add Webhook Response for frontend apps             |
| NOTE — Overview      | Sticky Note          | Documentation / overview        | None                 | None                 | This workflow integrates BananaAPI.com with the Nano Banana engine for cost-effective image generation                 |
| NOTE — Prerequisites | Sticky Note          | Documentation / prerequisites   | None                 | None                 | Requires BananaAPI account and token, n8n instance, prompt knowledge; keep API keys secure                             |
| NOTE — Step 1 (Form Input) | Sticky Note     | Documentation of form step      | None                 | None                 | Details input fields and purpose                                                                                      |
| NOTE — Step 2 (POST Request) | Sticky Note    | Documentation of API submission | None                 | None                 | Details POST request setup                                                                                            |
| NOTE — Step 3 (Processing) | Sticky Note      | Documentation of status polling | None                 | None                 | Explains polling logic and timings                                                                                    |
| NOTE — Step 5 (Output) | Sticky Note         | Documentation of output step    | None                 | None                 | Explains final data returned                                                                                          |
| NOTE — API Reference  | Sticky Note          | Documentation of API endpoints  | None                 | None                 | Lists POST and GET API endpoints and header requirements                                                             |
| NOTE — Pricing Advantages | Sticky Note       | Documentation of pricing        | None                 | None                 | Highlights cost-effectiveness and pay-as-you-go pricing                                                               |
| NOTE — Troubleshooting | Sticky Note         | Documentation of error handling | None                 | None                 | Lists common errors and troubleshooting tips                                                                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node:**  
   - Name: `Form — Get Prompt`  
   - Type: Form Trigger (v2.2)  
   - Configure form title: "Banana API — Image Generator"  
   - Add form fields:  
     - `api_token`: string, required, placeholder "Bearer xxxxxxx..."  
     - `prompt`: textarea, required, placeholder example "a neon cyberpunk cat, detailed, 4k"  
     - `Output Format [optional]`: dropdown, options `png`, `jpeg`  
     - `Image Size [optional]`: dropdown, options `16:9`, `9:16`, `1:1`, `3:4`, `4:3`  
     - `image_url_1 [optional]` through `image_url_5 [optional]`: string URL fields, optional  
   - Set form description accordingly.

2. **Create HTTP Request Node for Submission:**  
   - Name: `Submit — Banana API`  
   - Type: HTTP Request (v4.2)  
   - Method: POST  
   - URL: `https://bananaapi.com/api/n8n/generate/`  
   - Headers:  
     - `Content-Type`: `application/json`  
     - `Authorization`: Expression `Bearer {{$json.api_token}}` (from form input)  
   - Body: JSON type, raw JSON with expressions to insert form data:  
     ```json
     {
       "input": {
         "prompt": "{{$json.prompt}}",
         "image_size": "{{$json['Image Size [optional]']}}",
         "output_format": "{{$json['Output Format [optional]']}}",
         "image_urls": [
           "{{$json['image_url_1 [optional]']}}",
           "{{$json['image_url_2 [optional]']}}",
           "{{$json['image_url_3 [optional]']}}",
           "{{$json['image_url_4 [optional]']}}",
           "{{$json['image_url_5 [optional]']}}"
         ]
       }
     }
     ```
   - Connect output of Form Trigger to this node.

3. **Create Wait Node:**  
   - Name: `Wait 5s`  
   - Type: Wait (v1.1)  
   - Default 5 seconds wait.  
   - Connect output of Submit — Banana API to this node.

4. **Create HTTP Request Node for Status Check:**  
   - Name: `Check Status`  
   - Type: HTTP Request (v4.2)  
   - Method: GET (default)  
   - URL: Expression:  
     `https://bananaapi.com/api/n8n/image-status/{{$('Submit — Banana API').item.json.taskId}}`  
   - Connect output of Wait 5s to this node.

5. **Create Conditional Node:**  
   - Name: `If`  
   - Type: If (v2.2)  
   - Condition:  
     - Left: Expression `{{$('Check Status').item.json.status}}`  
     - Operator: equals  
     - Right: `completed`  
   - Connect output of Check Status to this node.

6. **Create Wait Node for Loop:**  
   - Name: `Wait 5s (loop)`  
   - Type: Wait (v1.1)  
   - Default 5 seconds wait.  
   - Connect false branch (status != completed) of If node to this node.  
   - Connect output of this wait node back to Check Status node to form polling loop.

7. **Create Set Node for Output:**  
   - Name: `Return Links`  
   - Type: Set (v3.4)  
   - Assign variables:  
     - `image_url` = expression `{{$json.imageUrl}}` (from Check Status response)  
     - `task_id` = expression `{{$('Submit — Banana API').item.json.taskId}}`  
     - `status` = expression `{{$json.status}}`  
   - Connect true branch (status == completed) of If node to this node.

8. **Optional:**  
   - Add Webhook Response node after `Return Links` to send JSON back to frontend clients if desired.  
   - Store API token securely as n8n Credential for production.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                    | Context or Link                         |
|-------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------|
| This workflow integrates BananaAPI.com with the Nano Banana image generation engine for cost-effective AI image creation.                      | Overview Sticky Note                   |
| Requires BananaAPI.com account + API key (Bearer token). Create key at https://bananaapi.com/api                                                | Prerequisites Sticky Note              |
| Pay only $0.025 per image, no monthly subscription, credits never expire.                                                                        | Pricing Advantages Sticky Note         |
| API endpoints used: POST https://bananaapi.com/api/n8n/generate/, GET https://bananaapi.com/api/n8n/image-status/{taskId}                       | API Reference Sticky Note              |
| Common errors: 401/403 Unauthorized, invalid JSON, missing imageUrl, slow performance; increase wait time if needed.                            | Troubleshooting Sticky Note            |
| For best results, provide clear, descriptive prompts and optionally up to 5 reference images via URLs.                                           | Form Input Sticky Note                 |
| Tip: add Webhook Response node after Return Links for clean JSON response to frontend applications.                                              | Output Sticky Note                     |
| Documentation and API details: https://bananaapi.com/api                                                                                         | API Reference Sticky Note              |

---

**Disclaimer:** The provided text is exclusively extracted from an automated n8n workflow integration. It complies with all current content policies and contains no illegal, offensive, or protected content. All data processed is legal and public.