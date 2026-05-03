Generate Product Creative Images with Google Nano-Banana Model via Defapi

https://n8nworkflows.xyz/workflows/generate-product-creative-images-with-google-nano-banana-model-via-defapi-8484


# Generate Product Creative Images with Google Nano-Banana Model via Defapi

### 1. Workflow Overview

This n8n workflow is designed to generate AI-powered product creative images by leveraging the Defapi API with Google's Nano-Banana model. It enables users, such as marketers, product designers, e-commerce professionals, and content creators, to submit a text prompt describing a creative scene along with a product image URL and their API key. The workflow automates the image generation request, monitors the generation status through polling, and finally retrieves and presents the generated creative image. It also provides a feature to check the user's current credit balance on Defapi.

The workflow is logically divided into the following blocks:

- **1.1 User Input Reception (Form Trigger)**: Collects user inputs via a web form.
- **1.2 Image Generation Request Submission**: Sends the creative generation request to Defapi.
- **1.3 Image Processing Status Polling**: Waits and repeatedly checks the generation status until completion.
- **1.4 Result Formatting and Display**: Extracts and formats the generated image URL for output.
- **1.5 User Credit Balance Retrieval**: Fetches and shows the user's current Defapi credit balance.
- **1.6 Documentation and Visual Aids (Sticky Notes)**: Provides workflow overview, instructions, and example images for user reference.

---

### 2. Block-by-Block Analysis

#### 1.1 User Input Reception (Form Trigger)

- **Overview:**  
  This block initiates the workflow by presenting a form to the user to collect the text prompt, product image URL, and API key required for image generation.

- **Nodes Involved:**  
  - Submit Image for Creative Generation

- **Node Details:**

  - **Submit Image for Creative Generation**  
    - Type: Form Trigger  
    - Role: Serves as the entry point, presenting a web form to collect user inputs.  
    - Configuration:  
      - Form title: "AI Product Creative Generator"  
      - Form description: "Please fill in the following information to generate your creative."  
      - Fields:  
        - `prompt`: Text input with placeholder "Generate a gorgeous scene for this product for advertising creative"  
        - `img_url`: Text input with placeholder URL example  
        - `api_key`: Text input for the Defapi API key (masked in placeholder for security)  
      - Webhook ID: Provided for integration and triggering  
    - Input/Output:  
      - Input: HTTP form submission  
      - Output: JSON containing submitted data (`prompt`, `img_url`, `api_key`)  
    - Edge Cases / Failures:  
      - Missing or invalid API key results in authorization failures downstream.  
      - Invalid or missing prompt/image URL inputs may cause API request errors or poor image generation quality.

#### 1.2 Image Generation Request Submission

- **Overview:**  
  Sends a POST request to Defapi's image generation endpoint with the user-provided prompt, product image URL, and API key.

- **Nodes Involved:**  
  - Send Image Generation Request to Defapi.org API

- **Node Details:**

  - **Send Image Generation Request to Defapi.org API**  
    - Type: HTTP Request  
    - Role: Submits the creative image generation job to the Defapi API using Google's Nano-Banana model.  
    - Configuration:  
      - Method: POST  
      - URL: `https://api.defapi.org/api/image/gen`  
      - Headers:  
        - Content-Type: application/json  
        - Authorization: Bearer token from input `api_key`  
      - Body (JSON):  
        ```json
        {
          "prompt": "<user prompt>",
          "model": "google/nano-banana",
          "images": ["<product image URL>"]
        }
        ```  
      - Uses expressions to dynamically insert user inputs from the form trigger item (`$json.prompt`, `$json.img_url`, `$json.api_key`).  
    - Input/Output:  
      - Input: User form data  
      - Output: JSON response containing task ID and status  
    - Edge Cases / Failures:  
      - Authorization failure if API key is invalid or expired.  
      - API endpoint errors or timeouts.  
      - Invalid prompt or image URL may cause generation errors.

#### 1.3 Image Processing Status Polling

- **Overview:**  
  This block implements a wait-and-check loop: it waits 10 seconds, then queries the API for the status of the image generation task. It repeats until the task status is "success".

- **Nodes Involved:**  
  - Wait for Image Processing Completion  
  - Obtain the generated status  
  - Check if Image Generation is Complete

- **Node Details:**

  - **Wait for Image Processing Completion**  
    - Type: Wait  
    - Role: Pauses the workflow for 10 seconds between status checks to avoid excessive API calls.  
    - Configuration: Wait 10 seconds.  
    - Input/Output: Receives the task ID and passes it downstream without modification.  
    - Edge Cases: Long processing times may cause delays; consider timeout or max retries if needed.

  - **Obtain the generated status**  
    - Type: HTTP Request  
    - Role: Queries Defapi API for the current status of the image generation task.  
    - Configuration:  
      - Method: GET  
      - URL: `https://api.defapi.org/api/task/query`  
      - Query Parameter: `task_id` from previous API response (`{{$json.data.task_id}}`)  
      - Headers:  
        - Content-Type: application/json  
        - Authorization: Bearer token from original API key input (`$('Submit Image for Creative Generation').item.json.api_key`)  
    - Input/Output: Takes task ID, outputs status JSON.  
    - Edge Cases:  
      - Authorization errors if token invalid.  
      - API errors or network timeouts.

  - **Check if Image Generation is Complete**  
    - Type: If  
    - Role: Evaluates whether the image generation task status equals "success".  
    - Configuration:  
      - Condition: `{{$json.data.status == 'success'}}` equals `true`  
    - Input/Output:  
      - If true: proceeds to format/display results.  
      - If false: loops back to wait node for another polling cycle.  
    - Edge Cases: Infinite loop risk if task never completes; consider implementing a max retry limit or error path.

#### 1.4 Result Formatting and Display

- **Overview:**  
  Extracts the generated image URL from API response and prepares it for output display and further use.

- **Nodes Involved:**  
  - Format and Display Image Results

- **Node Details:**

  - **Format and Display Image Results**  
    - Type: Set  
    - Role: Sets a new JSON field `image_url` with the first generated image URL from the API response.  
    - Configuration:  
      - Assigns `image_url` to `{{$json.data.result[0].image}}`  
    - Input/Output:  
      - Input: Status response with image data  
      - Output: JSON containing `image_url` for downstream consumption or display  
    - Edge Cases:  
      - If API returns empty or malformed `result` array, this may fail or return undefined.  
      - Should validate presence of image URL.

#### 1.5 User Credit Balance Retrieval

- **Overview:**  
  Retrieves and displays the user's current credit balance from Defapi to inform them of usage capacity.

- **Nodes Involved:**  
  - Get Your Balance  
  - Show Balance

- **Node Details:**

  - **Get Your Balance**  
    - Type: HTTP Request  
    - Role: Calls Defapi user endpoint to get account info including credit balance.  
    - Configuration:  
      - Method: GET  
      - URL: `https://api.defapi.org/api/user`  
      - Headers:  
        - Authorization: Bearer token from original API key input  
    - Input/Output:  
      - Input: none, triggered after image URL retrieval  
      - Output: JSON with user credit data  
    - Edge Cases:  
      - Authorization failures if API key invalid.  
      - Network or API errors.

  - **Show Balance**  
    - Type: Set  
    - Role: Extracts the `credit` field from user data and formats it in JSON under `data.credit`.  
    - Configuration:  
      - Assigns `data.credit` to `{{$json.data.credit}}`  
    - Input/Output:  
      - Input: User data JSON  
      - Output: JSON with formatted credit info for display or further use

#### 1.6 Documentation and Visual Aids (Sticky Notes)

- **Overview:**  
  Provides user and maintainer references within the workflow, including overview instructions, example images, and branding.

- **Nodes Involved:**  
  - Sticky Note3 (Overview & Setup Instructions)  
  - Sticky Note2 (Product Image example)  
  - Sticky Note4 (Product Creative example)

- **Node Details:**

  - **Sticky Note3**  
    - Type: Sticky Note  
    - Role: Contains the comprehensive workflow overview, prerequisites, setup instructions, and customization tips.  
    - Content: Markdown formatted text, including links to Defapi.org, usage instructions, and tips for improving prompt quality.

  - **Sticky Note2**  
    - Type: Sticky Note  
    - Role: Displays an example product image used for creative generation.  
    - Content: Markdown with embedded image link: `https://i.imgur.com/ScdsEr2.png`

  - **Sticky Note4**  
    - Type: Sticky Note  
    - Role: Displays an example of the generated product creative image.  
    - Content: Markdown with embedded image link: `https://i.imgur.com/KJiKE3i.png`

---

### 3. Summary Table

| Node Name                           | Node Type         | Functional Role                                      | Input Node(s)                        | Output Node(s)                         | Sticky Note                                                                                                    |
|-----------------------------------|-------------------|-----------------------------------------------------|------------------------------------|--------------------------------------|---------------------------------------------------------------------------------------------------------------|
| Submit Image for Creative Generation | Form Trigger      | Collect user inputs: prompt, image URL, API key     | -                                  | Send Image Generation Request to Defapi.org API |                                                                                                               |
| Send Image Generation Request to Defapi.org API | HTTP Request     | Submit image generation job to Defapi API           | Submit Image for Creative Generation | Wait for Image Processing Completion |                                                                                                               |
| Wait for Image Processing Completion | Wait              | Pause 10 seconds before polling status              | Send Image Generation Request to Defapi.org API | Obtain the generated status            |                                                                                                               |
| Obtain the generated status        | HTTP Request      | Query Defapi for current task status                 | Wait for Image Processing Completion | Check if Image Generation is Complete |                                                                                                               |
| Check if Image Generation is Complete | If                | Check if generation status is "success"              | Obtain the generated status          | Format and Display Image Results / Wait for Image Processing Completion |                                                                                                               |
| Format and Display Image Results   | Set               | Extract and set generated image URL                  | Check if Image Generation is Complete | Get Your Balance                      |                                                                                                               |
| Get Your Balance                   | HTTP Request      | Retrieve user credit balance from Defapi             | Format and Display Image Results     | Show Balance                         |                                                                                                               |
| Show Balance                      | Set               | Format user credit for output                         | Get Your Balance                    | -                                    |                                                                                                               |
| Sticky Note3                     | Sticky Note       | Workflow overview, setup instructions, customization tips | -                                  | -                                    | Contains extensive setup instructions and usage notes with link: https://defapi.org                            |
| Sticky Note2                     | Sticky Note       | Example product image                                 | -                                  | -                                    | Product Image example: https://i.imgur.com/ScdsEr2.png                                                        |
| Sticky Note4                     | Sticky Note       | Example product creative image                        | -                                  | -                                    | Product Creative example: https://i.imgur.com/KJiKE3i.png                                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger node: "Submit Image for Creative Generation"**  
   - Type: Form Trigger  
   - Configure webhook ID (auto-generated or custom)  
   - Set form title: "AI Product Creative Generator"  
   - Set form description: "Please fill in the following information to generate your creative."  
   - Add form fields:  
     - Field 1: Label `prompt`, placeholder "Generate a gorgeous scene for this product for advertising creative"  
     - Field 2: Label `img_url`, placeholder example URL (e.g., https://cdn.openai.com/API/docs/images/body-lotion.png)  
     - Field 3: Label `api_key`, placeholder masked API key (e.g., dk-087cc3********************)  
   - Save node.

2. **Create HTTP Request node: "Send Image Generation Request to Defapi.org API"**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.defapi.org/api/image/gen`  
   - Headers:  
     - Content-Type: application/json  
     - Authorization: `Bearer {{$json.api_key}}` (use expression to use the API key from input)  
   - Body Type: JSON (set `specifyBody` to JSON)  
   - Body:  
     ```json
     {
       "prompt": "{{$json.prompt}}",
       "model": "google/nano-banana",
       "images": ["{{$json.img_url}}"]
     }
     ```  
   - Connect the output of the Form Trigger node to this node.

3. **Create Wait node: "Wait for Image Processing Completion"**  
   - Type: Wait  
   - Configure to wait 10 seconds  
   - Connect output of HTTP Request node (image generation) to this node.

4. **Create HTTP Request node: "Obtain the generated status"**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://api.defapi.org/api/task/query`  
   - Query Parameters:  
     - `task_id` with value `={{$json.data.task_id}}` (expression from previous response)  
   - Headers:  
     - Content-Type: application/json  
     - Authorization: `Bearer {{ $('Submit Image for Creative Generation').item.json.api_key }}` (expression referencing original API key)  
   - Connect Wait node output to this status check node.

5. **Create If node: "Check if Image Generation is Complete"**  
   - Type: If  
   - Condition:  
     - Check if expression `{{$json.data.status == 'success'}}` equals `true`  
   - Connect output of status check node to this If node.

6. **Loop for polling:**  
   - If condition false (image not ready): connect back to Wait node to repeat polling.  
   - If condition true (image ready): connect to next node for result formatting.

7. **Create Set node: "Format and Display Image Results"**  
   - Type: Set  
   - Assign field `image_url` with expression `{{$json.data.result[0].image}}`  
   - Connect If node true output to this node.

8. **Create HTTP Request node: "Get Your Balance"**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://api.defapi.org/api/user`  
   - Headers:  
     - Authorization: `Bearer {{ $('Submit Image for Creative Generation').item.json.api_key }}`  
   - Connect output of formatting node to this node.

9. **Create Set node: "Show Balance"**  
   - Type: Set  
   - Assign field `data.credit` with expression `{{$json.data.credit}}`  
   - Connect output of Get Your Balance node to this node.

10. **Add Sticky Notes for documentation and visuals:**  
    - Create Sticky Note with overview and setup instructions content as per the Sticky Note3 content.  
    - Create Sticky Note with example product image (Markdown embedding image from https://i.imgur.com/ScdsEr2.png).  
    - Create Sticky Note with example generated creative image (Markdown embedding image from https://i.imgur.com/KJiKE3i.png).

11. **Final Checks:**  
    - Ensure all connections are accurate and nodes have the correct parameter expressions.  
    - Test workflow by executing and submitting a sample form input.  
    - Validate error handling for invalid API keys or inputs.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                          | Context or Link                        |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------|
| This workflow requires a Defapi account and valid API key. Registration and API key generation are available at [Defapi.org](https://defapi.org). Keep your API key secure and do not share publicly.                                                                                                                                                                                                   | https://defapi.org                    |
| Optimize AI prompt quality by including descriptive elements such as scene setting, lighting, style (realistic, artistic, cinematic), product placement, and visual composition for the best creative image results.                                                                                                                                                                                     | Customization recommendation         |
| The workflow uses Google's Nano-Banana model, a specialized AI model tailored for generating product creative images.                                                                                                                                                                                                                                                                               | Model specification                  |
| For long-running image generation tasks, consider implementing additional timeout or maximum retry logic in the polling loop to avoid infinite waiting.                                                                                                                                                                                                                                              | Suggested enhancement                |
| Example product and creative images are embedded as sticky notes within the workflow for visual reference.                                                                                                                                                                                                                                                                                           | https://i.imgur.com/ScdsEr2.png, https://i.imgur.com/KJiKE3i.png |
| The workflow is compatible with n8n version 0.190.0+ due to use of HTTP Request node version 4.2 and expression syntax. Verify node version compatibility when upgrading or modifying.                                                                                                                                                                                                              | Version requirement                  |

---

**Disclaimer:**  
The provided text and workflow originate exclusively from an automated n8n workflow designed for lawful and public data handling. The workflow follows all applicable content policies and contains no illegal, offensive, or protected material.