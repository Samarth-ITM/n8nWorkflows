Generate Graphic Wallpaper with Midjourney, GPT-4o-mini and Canvas APIs

https://n8nworkflows.xyz/workflows/generate-graphic-wallpaper-with-midjourney--gpt-4o-mini-and-canvas-apis-3627


# Generate Graphic Wallpaper with Midjourney, GPT-4o-mini and Canvas APIs

### 1. Workflow Overview

This workflow is designed to automate the creation of visually compelling graphic wallpapers by integrating AI text generation, image generation, and graphic design APIs. It targets content creators, social media professionals, marketing teams, blog authors, and knowledge-based creators who want to produce artistic visual posts, promotional graphics, featured images, or shareable card visuals efficiently.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Parameter Setup:** Collects user-defined parameters including API keys, theme, scenario, style, example texts, and image prompts.
- **1.2 AI Text Generation (GPT-4o-mini):** Generates a creative prompt sentence based on the input parameters.
- **1.3 Prompt Preparation:** Combines the AI-generated text with the image prompt to create a final prompt for image generation.
- **1.4 Image Generation (Midjourney via PiAPI):** Sends the prompt to Midjourney API to generate an image and polls for completion.
- **1.5 Image URL Extraction and Status Checking:** Extracts the generated image URL and verifies the generation status.
- **1.6 Final Design Composition (Canvas API):** Uses the Canvas API to compose the final graphic wallpaper by combining the generated image and text into a design template.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Parameter Setup

- **Overview:**  
  This block initializes the workflow by receiving manual trigger input and setting up essential parameters including the PiAPI key and descriptive inputs for theme, scenario, style, examples, and image prompt.

- **Nodes Involved:**  
  - When clicking Test workflow  
  - Basic Params  
  - Sticky Note (Basic Params instructions)

- **Node Details:**

  - **When clicking Test workflow**  
    - Type: Manual Trigger  
    - Role: Starts the workflow manually for testing or execution.  
    - Configuration: No parameters; triggers downstream nodes.  
    - Inputs: None  
    - Outputs: Connects to Basic Params node.  
    - Edge cases: None specific; manual trigger requires user action.

  - **Basic Params**  
    - Type: Set  
    - Role: Defines and stores user inputs and API key for subsequent nodes.  
    - Configuration: Raw JSON with fields:  
      - `x-api-key` (string, empty by default, must be filled by user)  
      - `theme` (e.g., "Hope")  
      - `scenario` (e.g., "Don't know about the future...")  
      - `style` (e.g., "Cinematic Grandeur, Sci-Tech Aesthetic, 3D style")  
      - `example` (multi-sentence examples for style guidance)  
      - `image_prompt` (detailed description for image generation)  
    - Inputs: From manual trigger  
    - Outputs: To Gpt-4o-mini API node  
    - Edge cases: Missing or invalid API key will cause authentication errors downstream.

  - **Sticky Note1**  
    - Type: Sticky Note  
    - Role: Provides user instructions on filling Basic Params node.  
    - Content: Explains the meaning of each parameter and where to get the API key.  
    - Inputs/Outputs: None (informational only).

#### 2.2 AI Text Generation (GPT-4o-mini)

- **Overview:**  
  Generates a creative, style-specific prompt sentence based on the theme, scenario, style, and example texts provided by the user.

- **Nodes Involved:**  
  - Gpt-4o-mini API

- **Node Details:**

  - **Gpt-4o-mini API**  
    - Type: HTTP Request  
    - Role: Calls PiAPI's GPT-4o-mini chat completion endpoint to generate a prompt sentence.  
    - Configuration:  
      - POST to `https://api.piapi.ai/v1/chat/completions`  
      - JSON body includes a system message ("You are a helpful assistant.") and a user message constructed dynamically using expressions referencing `style`, `theme`, `scenario`, and `example` from Basic Params.  
      - Authorization header uses Bearer token from `x-api-key`.  
    - Inputs: From Basic Params  
    - Outputs: To Get Prompt node  
    - Edge cases:  
      - Authentication failure if API key is invalid or missing.  
      - API rate limits or downtime.  
      - Expression failures if input parameters are missing or malformed.

#### 2.3 Prompt Preparation

- **Overview:**  
  Combines the AI-generated prompt sentence with the user-defined image prompt to create a final prompt string for image generation.

- **Nodes Involved:**  
  - Get Prompt

- **Node Details:**

  - **Get Prompt**  
    - Type: Code (JavaScript)  
    - Role: Processes outputs from GPT-4o-mini API and Basic Params to produce a combined prompt.  
    - Configuration:  
      - Extracts `image_prompt` from Basic Params.  
      - Extracts AI-generated prompt sentence from GPT-4o-mini API response.  
      - Replaces placeholder `'xxx'` in `image_prompt` with the AI-generated prompt.  
      - Returns an object containing `show_prompt` (AI-generated text) and `prompt` (final combined prompt).  
    - Inputs: From Gpt-4o-mini API and Basic Params (via expression)  
    - Outputs: To Midjourney Generator node  
    - Edge cases:  
      - If AI-generated prompt is missing or malformed, replacement may fail.  
      - If `image_prompt` does not contain `'xxx'`, replacement has no effect.

#### 2.4 Image Generation (Midjourney via PiAPI)

- **Overview:**  
  Sends the final prompt to Midjourney API to generate an image, then waits and polls until the image generation is complete or failed.

- **Nodes Involved:**  
  - Midjourney Generator  
  - Wait for Midjourney Generation  
  - Get Midjourney Task  
  - Determine Whether the Image URL was Fetched  
  - Check Image Generation Status

- **Node Details:**

  - **Midjourney Generator**  
    - Type: HTTP Request  
    - Role: Initiates an image generation task on Midjourney via PiAPI.  
    - Configuration:  
      - POST to `https://api.piapi.ai/api/v1/task`  
      - JSON body includes:  
        - model: "midjourney"  
        - task_type: "imagine"  
        - input: prompt (from Get Prompt), aspect_ratio "1:1", process_mode "turbo", skip_prompt_check false  
      - Header `x-api-key` from Basic Params.  
    - Inputs: From Get Prompt  
    - Outputs: To Wait for Midjourney Generation  
    - Edge cases:  
      - API key errors, network issues, invalid prompt errors.

  - **Wait for Midjourney Generation**  
    - Type: Wait  
    - Role: Pauses workflow to allow image generation to complete asynchronously.  
    - Configuration: Default wait with webhook ID (used for resuming).  
    - Inputs: From Midjourney Generator  
    - Outputs: To Get Midjourney Task  
    - Edge cases: Timeout if generation takes too long or webhook fails.

  - **Get Midjourney Task**  
    - Type: HTTP Request  
    - Role: Polls Midjourney task status by task_id to check if image generation is complete.  
    - Configuration:  
      - GET to `https://api.piapi.ai/api/v1/task/{{ $json.data.task_id }}`  
      - Header `x-api-key` from Basic Params.  
    - Inputs: From Wait for Midjourney Generation  
    - Outputs: To Determine Whether the Image URL was Fetched  
    - Edge cases:  
      - Task ID missing or expired.  
      - API errors or rate limits.

  - **Determine Whether the Image URL was Fetched**  
    - Type: If  
    - Role: Checks if the task status is either "completed" or "failed".  
    - Configuration:  
      - Condition: `$json.data.status` equals "completed" OR "failed"  
    - Inputs: From Get Midjourney Task  
    - Outputs:  
      - True branch: To Check Image Generation Status  
      - False branch: Loops back to Wait for Midjourney Generation (polling)  
    - Edge cases: Infinite loop if status never changes.

  - **Check Image Generation Status**  
    - Type: Switch  
    - Role: Routes workflow based on task status.  
    - Configuration:  
      - Routes only "completed" status forward.  
      - Other statuses implicitly ignored or halted.  
    - Inputs: From Determine Whether the Image URL was Fetched  
    - Outputs: To Get Image Url  
    - Edge cases: No explicit failure handling for "failed" status.

#### 2.5 Image URL Extraction and Status Checking

- **Overview:**  
  Extracts the generated image URLs from the Midjourney task output and prepares them for the final design step.

- **Nodes Involved:**  
  - Get Image Url

- **Node Details:**

  - **Get Image Url**  
    - Type: Set  
    - Role: Extracts and sets image URL fields from the Midjourney API response for downstream use.  
    - Configuration:  
      - Assigns `data.output.temporary_image_urls` (array) and `data.output.image_url` (string) from input JSON.  
    - Inputs: From Check Image Generation Status  
    - Outputs: To Design in Canvas  
    - Edge cases: Missing or empty image URLs if generation failed or incomplete.

#### 2.6 Final Design Composition (Canvas API)

- **Overview:**  
  Uses the Canvas Switchboard API to compose the final graphic wallpaper by combining the generated image and AI-generated text into a predefined design template.

- **Nodes Involved:**  
  - Design in Canvas  
  - Sticky Note2 (Canvas API instructions)

- **Node Details:**

  - **Design in Canvas**  
    - Type: HTTP Request  
    - Role: Sends a design request to Canvas Switchboard API to create a graphic wallpaper.  
    - Configuration:  
      - POST to `https://api.canvas.switchboard.ai`  
      - JSON body includes:  
        - `template`: "social-3-1" (predefined template name)  
        - `sizes`: width 1000, height 1500  
        - `elements`:  
          - `text1`: text content from `show_prompt` (AI-generated prompt), with semicolons replaced by newline characters for formatting  
          - `rectangle1`: fillColor set to white (#fff)  
          - `image1`: URL from Midjourney generated image (`temporary_image_urls[0]`)  
      - Header `X-API-Key` hardcoded as "45ba3916-2f10-497d-815b-7ffc9b69001f" (likely a Canvas API key)  
    - Inputs: From Get Image Url  
    - Outputs: Final output of the workflow (image design result)  
    - Edge cases:  
      - API key invalid or expired.  
      - Template name mismatch or missing elements.  
      - Network or API errors.

  - **Sticky Note2**  
    - Type: Sticky Note  
    - Role: Provides instructions and tips for using the Canvas API node, including template design and customization.  
    - Inputs/Outputs: None (informational only).

---

### 3. Summary Table

| Node Name                         | Node Type           | Functional Role                          | Input Node(s)                   | Output Node(s)                   | Sticky Note                                                                                                         |
|----------------------------------|---------------------|----------------------------------------|--------------------------------|---------------------------------|---------------------------------------------------------------------------------------------------------------------|
| Sticky Note                      | Sticky Note         | Workflow introduction and purpose      | None                           | None                            | ## Generate Graphic Wallpaper with Midjourney, GPT-4o-mini and Canvas APIs We design this workflow with PiAPI APIs and Canvas API with the purpose to produce a visually compelling image with resonant copy to spark emotional connection. 🙌 Wish you make a fantastic generation with our workflow! |
| When clicking Test workflow      | Manual Trigger      | Starts the workflow manually            | None                           | Basic Params                    |                                                                                                                     |
| Basic Params                    | Set                 | Defines user inputs and API key         | When clicking Test workflow    | Gpt-4o-mini API                | ## Basic Params In **Basic Params** node, you need to fill in your PiAPI key which you could check in [My Account](https://piapi.ai/workspace/my-account) on [PiAPI](https://piapi.ai). Other information you need to provide is listed as follow📝: 1. Theme. The theme refers to the topic that you want to talk about when you start your generation. 2. Scenario. The scenario text usually describe your status about your feeling. 3. Style. The style of the image. 4. Example. The text example shown to LLM to describe a style t you want to generate. 5. Image prompt. Image prompt is usually about the context of the image that you want to generate. |
| Gpt-4o-mini API                 | HTTP Request        | Generates creative prompt sentence      | Basic Params                  | Get Prompt                    |                                                                                                                     |
| Get Prompt                     | Code                | Combines AI prompt with image prompt    | Gpt-4o-mini API               | Midjourney Generator          |                                                                                                                     |
| Midjourney Generator           | HTTP Request        | Initiates image generation task         | Get Prompt                   | Wait for Midjourney Generation |                                                                                                                     |
| Wait for Midjourney Generation | Wait                | Pauses workflow awaiting image completion | Midjourney Generator          | Get Midjourney Task           |                                                                                                                     |
| Get Midjourney Task            | HTTP Request        | Polls image generation task status      | Wait for Midjourney Generation | Determine Whether the Image URL was Fetched |                                                                                                                     |
| Determine Whether the Image URL was Fetched | If                  | Checks if image generation is complete or failed | Get Midjourney Task           | Check Image Generation Status (true), Wait for Midjourney Generation (false) |                                                                                                                     |
| Check Image Generation Status  | Switch              | Routes based on image generation status | Determine Whether the Image URL was Fetched | Get Image Url                |                                                                                                                     |
| Get Image Url                 | Set                 | Extracts generated image URLs            | Check Image Generation Status | Design in Canvas             |                                                                                                                     |
| Design in Canvas              | HTTP Request        | Creates final graphic wallpaper design  | Get Image Url                 | None                        | ## Design in Canvas API Node We make a final design with Canvas API. You could check the node code to make a template design more efficiently in Canvas. Also you could make various artworks with template library in Canvas. You could modify node parameters to or add more nodes to make more artworks in one role. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node**  
   - Name: "When clicking Test workflow"  
   - Purpose: To manually start the workflow.

2. **Create a Set node**  
   - Name: "Basic Params"  
   - Purpose: Store user inputs and API key.  
   - Configuration:  
     - JSON mode with fields:  
       - `x-api-key`: (string, leave empty for user input)  
       - `theme`: e.g., "Hope"  
       - `scenario`: e.g., "Don't know about the future, confused and feel lost about AI agent"  
       - `style`: e.g., "Cinematic Grandeur,Sci-Tech Aesthetic, 3D style"  
       - `example`: multi-line string with example sentences  
       - `image_prompt`: detailed image description with placeholder `'xxx'` for AI prompt insertion  
   - Connect "When clicking Test workflow" → "Basic Params"

3. **Create an HTTP Request node**  
   - Name: "Gpt-4o-mini API"  
   - Purpose: Generate a creative prompt sentence using GPT-4o-mini.  
   - Configuration:  
     - Method: POST  
     - URL: `https://api.piapi.ai/v1/chat/completions`  
     - Headers:  
       - Authorization: `Bearer {{ $json["x-api-key"] }}` (expression referencing Basic Params)  
     - Body (JSON):  
       ```json
       {
         "model": "gpt-4o-mini",
         "messages": [
           {"role": "system", "content": "You are a helpful assistant."},
           {"role": "user", "content": "Please {{ $json.style }}, based on the theme of {{ $json.theme }} and the scenario of {{ $json.scenario }}, according to the output case and language context. Examples are {{ $json.example }},Return a sentence directly, nothing else,Don't add a serial number, just a prompt that can be used for input,Do not add any quotation marks."}
         ]
       }
       ```  
   - Connect "Basic Params" → "Gpt-4o-mini API"

4. **Create a Code node**  
   - Name: "Get Prompt"  
   - Purpose: Combine AI-generated prompt with image prompt.  
   - Configuration (JavaScript):  
     ```javascript
     const image_prompt = $('Basic Params').first().json.image_prompt;
     const show_prompt = $input.first().json.choices[0].message.content;
     const prompt = image_prompt.replace(/'xxx'/, `'${show_prompt}'`);
     return { show_prompt, prompt };
     ```  
   - Connect "Gpt-4o-mini API" → "Get Prompt"

5. **Create an HTTP Request node**  
   - Name: "Midjourney Generator"  
   - Purpose: Submit image generation task to Midjourney via PiAPI.  
   - Configuration:  
     - Method: POST  
     - URL: `https://api.piapi.ai/api/v1/task`  
     - Headers:  
       - `x-api-key`: `{{ $('Basic Params').item.json['x-api-key'] }}`  
     - Body (JSON):  
       ```json
       {
         "model": "midjourney",
         "task_type": "imagine",
         "input": {
           "prompt": "{{ $json.prompt }}",
           "aspect_ratio": "1:1",
           "process_mode": "turbo",
           "skip_prompt_check": false
         }
       }
       ```  
   - Connect "Get Prompt" → "Midjourney Generator"

6. **Create a Wait node**  
   - Name: "Wait for Midjourney Generation"  
   - Purpose: Pause workflow to wait for image generation completion.  
   - Configuration: Use default wait with webhook ID.  
   - Connect "Midjourney Generator" → "Wait for Midjourney Generation"

7. **Create an HTTP Request node**  
   - Name: "Get Midjourney Task"  
   - Purpose: Poll Midjourney task status.  
   - Configuration:  
     - Method: GET  
     - URL: `https://api.piapi.ai/api/v1/task/{{ $json.data.task_id }}`  
     - Headers:  
       - `x-api-key`: `{{ $('Basic Params').item.json['x-api-key'] }}`  
   - Connect "Wait for Midjourney Generation" → "Get Midjourney Task"

8. **Create an If node**  
   - Name: "Determine Whether the Image URL was Fetched"  
   - Purpose: Check if task status is "completed" or "failed".  
   - Configuration:  
     - Condition: `$json.data.status` equals "completed" OR "failed"  
   - Connect "Get Midjourney Task" → "Determine Whether the Image URL was Fetched"  
   - True branch → "Check Image Generation Status"  
   - False branch → "Wait for Midjourney Generation" (loop for polling)

9. **Create a Switch node**  
   - Name: "Check Image Generation Status"  
   - Purpose: Route workflow based on status.  
   - Configuration:  
     - Route only if status is "completed"  
   - Connect "Determine Whether the Image URL was Fetched" (true) → "Check Image Generation Status"

10. **Create a Set node**  
    - Name: "Get Image Url"  
    - Purpose: Extract image URLs from Midjourney response.  
    - Configuration:  
      - Assign `data.output.temporary_image_urls` and `data.output.image_url` from input JSON.  
    - Connect "Check Image Generation Status" → "Get Image Url"

11. **Create an HTTP Request node**  
    - Name: "Design in Canvas"  
    - Purpose: Compose final wallpaper design using Canvas API.  
    - Configuration:  
      - Method: POST  
      - URL: `https://api.canvas.switchboard.ai`  
      - Headers:  
        - `X-API-Key`: `"45ba3916-2f10-497d-815b-7ffc9b69001f"` (Canvas API key)  
      - Body (JSON):  
        ```json
        {
          "template": "social-3-1",
          "sizes": [
            {
              "width": 1000,
              "height": 1500
            }
          ],
          "elements": {
            "text1": {
              "text": "{{ $('Get Prompt').item.json.show_prompt.replace(/;/g, \";\\n \")}}"
            },
            "rectangle1": {
              "fillColor": "#fff"
            },
            "image1": {
              "url": "{{ $json.data.output.temporary_image_urls[0] }}"
            }
          }
        }
        ```  
    - Connect "Get Image Url" → "Design in Canvas"

12. **Add Sticky Notes**  
    - Add a sticky note near Basic Params explaining parameter setup and API key instructions.  
    - Add a sticky note near Design in Canvas explaining template usage and customization.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                           | Context or Link                                                                                   |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| This workflow is designed for content creators and social media professionals to generate artistic visual posts using AI and design APIs.                                                                                          | Workflow purpose                                                                                 |
| Fill in your PiAPI API key from [PiAPI My Account](https://piapi.ai/workspace/my-account).                                                                                                                                           | API key setup                                                                                   |
| Set up a design template in [Canvas Switchboard](https://www.switchboard.ai) and obtain API key for Canvas API usage.                                                                                                              | Canvas API setup                                                                                |
| Example image prompt and style settings are provided to guide users in generating cinematic sci-fi themed images with Midjourney.                                                                                                   | Use case examples                                                                              |
| The workflow uses PiAPI's GPT-4o-mini and Midjourney models for text and image generation respectively, integrating them seamlessly with Canvas API for final design composition.                                                     | Technology stack                                                                               |
| For more design templates and customization, refer to Canvas Switchboard's template library and API documentation.                                                                                                                 | Canvas API resources                                                                           |
| The workflow includes manual trigger for testing and requires user input for API keys and parameters before execution.                                                                                                             | Execution instructions                                                                        |
| Potential failure points include API authentication errors, network timeouts, and infinite polling loops if image generation stalls. Proper error handling and monitoring are recommended for production use.                         | Error handling considerations                                                                |

---

This structured documentation enables advanced users and AI agents to fully understand, reproduce, and modify the workflow, anticipating integration points and potential failure modes.