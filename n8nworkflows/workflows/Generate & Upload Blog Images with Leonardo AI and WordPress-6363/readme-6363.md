Generate & Upload Blog Images with Leonardo AI and WordPress

https://n8nworkflows.xyz/workflows/generate---upload-blog-images-with-leonardo-ai-and-wordpress-6363


# Generate & Upload Blog Images with Leonardo AI and WordPress

### 1. Workflow Overview

This n8n workflow automates the process of generating blog images using Leonardo AI and uploading them to a WordPress site. It is designed for content creators or site administrators who want to streamline image creation and publication based on textual prompts. The workflow consists of two main logical blocks:

- **1.1 Image Generation with Leonardo AI**: Receives a textual prompt, sends it to Leonardo AI to generate an image, waits for the generation to complete, retrieves the image URL, and downloads the image data.

- **1.2 Upload to WordPress**: Takes the downloaded image, uploads it to WordPress media library with appropriate metadata, and extracts the uploaded image URL for further use.

The workflow supports two entry points: manual trigger for direct execution and an execution trigger to be called by other workflows with dynamic inputs. It includes retry mechanisms and waits to handle asynchronous image generation.

---

### 2. Block-by-Block Analysis

#### 2.1 Image Generation with Leonardo AI

**Overview:**  
This block handles sending the image prompt to Leonardo AI, polling the generation status after a delay, and fetching the generated image URL and binary data.

**Nodes Involved:**  
- When clicking ‘Execute workflow’ (Manual Trigger)  
- When Executed by Another Workflow (Execute Workflow Trigger)  
- Code1  
- HTTP Request1  
- Wait  
- HTTP Request2  
- HTTP Request3  

**Node Details:**

- **When clicking ‘Execute workflow’**  
  - *Type:* Manual Trigger  
  - *Role:* Starts the workflow manually, preloaded with example inputs (`prompt` and `slug`).  
  - *Config:* No parameters; triggers downstream nodes.  
  - *Inputs:* None  
  - *Outputs:* Connects to `Code1`  
  - *Edge cases:* No inputs may cause empty prompt errors downstream.

- **When Executed by Another Workflow**  
  - *Type:* Execute Workflow Trigger  
  - *Role:* Allows this workflow to be triggered with inputs (`prompt` and `slug`) from another workflow.  
  - *Config:* Defines required inputs `prompt` and `slug`.  
  - *Inputs:* External workflow execution  
  - *Outputs:* Connects to `Code1`  
  - *Edge cases:* Missing inputs or incorrect parameter names may cause errors.

- **Code1**  
  - *Type:* Code (JavaScript)  
  - *Role:* Passes all received input data downstream without modification (identity function).  
  - *Config:* `return $input.all();`  
  - *Inputs:* From either trigger node  
  - *Outputs:* Connects to `HTTP Request1`  
  - *Edge cases:* Expression failures unlikely; ensures data integrity.

- **HTTP Request1**  
  - *Type:* HTTP Request  
  - *Role:* Sends a POST request to Leonardo AI API to create an image generation job.  
  - *Config:*  
    - URL: `https://cloud.leonardo.ai/api/rest/v1/generations`  
    - Method: POST  
    - Body (JSON): Includes fixed modelId, contrast=3.5, styleUUID, dimensions (1472x832), enhancePrompt=true, number of images=1, and dynamic prompt from input JSON (`{{$json.prompt}}`).  
    - Headers: `content-type` and `accept` set to `application/json`  
    - Authentication: HTTP Bearer and HTTP Header Auth credentials for Leonardo AI  
  - Inputs: Receives prompt and slug from `Code1`  
  - Outputs: Connects to `Wait`  
  - Edge cases: Authentication failure, invalid prompt, API rate limits, network timeouts.

- **Wait**  
  - *Type:* Wait  
  - *Role:* Pauses workflow for 1 minute to allow Leonardo AI to process image generation asynchronously.  
  - *Config:* Wait time = 1 minute  
  - *Inputs:* From `HTTP Request1`  
  - *Outputs:* Connects to `HTTP Request2`  
  - Edge cases: Extended wait may be needed if generation is slow; no direct error.

- **HTTP Request2**  
  - *Type:* HTTP Request  
  - *Role:* Retrieves generation status and data from Leonardo AI using the generation ID received from `HTTP Request1`.  
  - *Config:*  
    - URL: `https://cloud.leonardo.ai/api/rest/v1/generations/{{ $json.sdGenerationJob.generationId }}` (dynamic)  
    - Method: GET (default)  
    - Headers: Accept JSON  
    - Authentication: Same Leonardo AI credentials  
  - Inputs: From `Wait`  
  - Outputs: Connects to `HTTP Request3`  
  - Edge cases: Generation not ready yet (may require additional wait or retries), invalid generationId, API errors.

- **HTTP Request3**  
  - *Type:* HTTP Request  
  - *Role:* Downloads the generated image file using the URL obtained from the previous step.  
  - *Config:*  
    - URL: `{{ $json.generations_by_pk.generated_images[0].url }}` (dynamic image URL)  
    - Method: GET  
    - Options: Default  
  - Inputs: From `HTTP Request2`  
  - Outputs: Connects to `Upload image2`  
  - Edge cases: URL invalid or expired, network errors, missing image.

---

#### 2.2 Upload to WordPress

**Overview:**  
This block uploads the downloaded image binary to a WordPress media library, setting the filename according to the blog slug, and extracts the uploaded image URL for further use.

**Nodes Involved:**  
- Upload image2  
- Code  

**Node Details:**

- **Upload image2**  
  - *Type:* HTTP Request  
  - *Role:* Uploads image binary data to WordPress REST API media endpoint.  
  - *Config:*  
    - URL: `https://your.wordpress.com/wp-json/wp/v2/media` (replace with actual WordPress domain)  
    - Method: POST  
    - Body: Binary data from previous node (`HTTP Request3`)  
    - Headers:  
      - `Content-Disposition: attachment; filename="img-{{ $('Code1').item.json.slug }}.jpg"` — dynamically sets filename using slug from input  
    - Authentication: Predefined WordPress OAuth2 credential named "Wordpress  - author@email.com"  
    - Retry on fail: Enabled (tries again after 5 seconds delay)  
  - Inputs: Binary image data from `HTTP Request3`, slug from `Code1`  
  - Outputs: Connects to `Code`  
  - Edge cases: Authentication failure, invalid token, upload size limits, network issues.

- **Code**  
  - *Type:* Code (JavaScript)  
  - *Role:* Extracts and returns the uploaded image URL from WordPress response JSON (field `guid.raw`).  
  - *Config:* `return {"image_url": $input.first().json.guid.raw}`  
  - *Inputs:* From `Upload image2`  
  - Outputs: None (end of workflow)  
  - Edge cases: If upload fails or response malformed, `guid.raw` may not exist causing errors.

---

### 3. Summary Table

| Node Name                 | Node Type               | Functional Role                     | Input Node(s)               | Output Node(s)         | Sticky Note                                                |
|---------------------------|-------------------------|-----------------------------------|-----------------------------|------------------------|------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger          | Manual start with preset inputs   | None                        | Code1                  |                                                            |
| When Executed by Another Workflow | Execute Workflow Trigger | External trigger with inputs      | None                        | Code1                  |                                                            |
| Code1                     | Code                    | Passes input data unchanged       | When clicking ‘Execute workflow’; When Executed by Another Workflow | HTTP Request1          |                                                            |
| HTTP Request1             | HTTP Request            | Sends image generation request to Leonardo AI | Code1                      | Wait                   | ## Generate image with leonardo                            |
| Wait                      | Wait                    | Waits 1 minute for generation     | HTTP Request1               | HTTP Request2           | ## Generate image with leonardo                            |
| HTTP Request2             | HTTP Request            | Polls Leonardo AI for generation results | Wait                       | HTTP Request3           | ## Generate image with leonardo                            |
| HTTP Request3             | HTTP Request            | Downloads generated image binary  | HTTP Request2               | Upload image2           | ## Generate image with leonardo                            |
| Upload image2             | HTTP Request            | Uploads image binary to WordPress | HTTP Request3               | Code                   | ## Upload to WordPress                                     |
| Code                      | Code                    | Extracts uploaded image URL       | Upload image2               | None                   | ## Upload to WordPress                                     |
| Sticky Note               | Sticky Note             | Visual note                      | None                        | None                   | ## Generate image with leonardo                            |
| Sticky Note1              | Sticky Note             | Visual note                      | None                        | None                   | ## Upload to WordPress                                     |
| Sticky Note2              | Sticky Note             | Visual note with example image    | None                        | None                   | ## Image generated ![batman-typing-on-a-laptop](https://articles.emp0.com/wp-content/uploads/2025/07/img-batman-typing-on-a-laptop.jpg) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Node type: Manual Trigger  
   - Name: `When clicking ‘Execute workflow’`  
   - Set pin data (optional):  
     - `slug`: "batman-typing-on-a-laptop"  
     - `prompt`: "Generate an image of batman typing on a laptop"  
   - Connect output to next node.

2. **Create Execute Workflow Trigger Node**  
   - Node type: Execute Workflow Trigger  
   - Name: `When Executed by Another Workflow`  
   - Define inputs `prompt` and `slug` as workflow parameters.  
   - Connect output to next node (`Code1`).

3. **Create Code Node**  
   - Node type: Code  
   - Name: `Code1`  
   - JavaScript code:  
     ```js
     return $input.all();
     ```  
   - Connect inputs from both triggers (`When clicking...` and `When Executed...`) into this node.  
   - Connect output to `HTTP Request1`.

4. **Create HTTP Request Node for Leonardo AI Generation**  
   - Node type: HTTP Request  
   - Name: `HTTP Request1`  
   - Method: POST  
   - URL: `https://cloud.leonardo.ai/api/rest/v1/generations`  
   - Authentication: Add HTTP Bearer and HTTP Header Auth credentials for Leonardo AI API.  
   - Headers:  
     - `content-type: application/json`  
     - `accept: application/json`  
   - Body (JSON):  
     ```json
     {
       "modelId": "b2614463-296c-462a-9586-aafdb8f00e36",
       "contrast": 3.5,
       "prompt": "{{$json.prompt}}",
       "num_images": 1,
       "width": 1472,
       "height": 832,
       "styleUUID": "111dc692-d470-4eec-b791-3475abac4c46",
       "enhancePrompt": true
     }
     ```  
   - Connect output to `Wait`.

5. **Create Wait Node**  
   - Node type: Wait  
   - Name: `Wait`  
   - Parameters: Wait for 1 minute  
   - Connect output to `HTTP Request2`.

6. **Create HTTP Request Node for Leonardo AI Status**  
   - Node type: HTTP Request  
   - Name: `HTTP Request2`  
   - Method: GET (default)  
   - URL: `https://cloud.leonardo.ai/api/rest/v1/generations/{{ $json.sdGenerationJob.generationId }}`  
   - Authentication: Same Leonardo AI credentials as `HTTP Request1`  
   - Header: `accept: application/json`  
   - Connect output to `HTTP Request3`.

7. **Create HTTP Request Node to Download Image**  
   - Node type: HTTP Request  
   - Name: `HTTP Request3`  
   - Method: GET  
   - URL: `{{ $json.generations_by_pk.generated_images[0].url }}`  
   - Connect output to `Upload image2`.

8. **Create HTTP Request Node to Upload Image to WordPress**  
   - Node type: HTTP Request  
   - Name: `Upload image2`  
   - Method: POST  
   - URL: `https://your.wordpress.com/wp-json/wp/v2/media` (replace domain with actual WordPress site)  
   - Authentication: Use predefined WordPress OAuth2 credential  
   - Headers:  
     - `Content-Disposition: attachment; filename="img-{{ $('Code1').item.json.slug }}.jpg"`  
   - Body content type: Binary data, input field name: `data`  
   - Retry on fail: Enabled with 5 seconds delay between retries  
   - Connect output to `Code`.

9. **Create Code Node to Extract Uploaded Image URL**  
   - Node type: Code  
   - Name: `Code`  
   - JavaScript code:  
     ```js
     return { "image_url": $input.first().json.guid.raw };
     ```  
   - This node marks the end of the workflow.

10. **Add Sticky Notes for Clarity**  
    - Create sticky notes with the following contents and place them close to related nodes:  
      - "## Generate image with leonardo" near nodes `HTTP Request1`, `Wait`, `HTTP Request2`, `HTTP Request3`.  
      - "## Upload to WordPress" near nodes `Upload image2`, `Code`.  
      - "## Image generated ![batman-typing-on-a-laptop](https://articles.emp0.com/wp-content/uploads/2025/07/img-batman-typing-on-a-laptop.jpg)" as a visual example.

---

### 5. General Notes & Resources

| Note Content                                                                                                    | Context or Link                                                                                           |
|----------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| Workflow uses Leonardo AI's specific model and style UUIDs; update these if Leonardo API changes or for customization | Leonardo AI API documentation (https://cloud.leonardo.ai/docs)                                           |
| WordPress upload URL must be replaced with your actual WordPress site's REST API media endpoint                 | WordPress REST API media endpoint docs: https://developer.wordpress.org/rest-api/reference/media/        |
| Retry on fail enabled on WordPress upload to handle intermittent network or auth issues                        | Improves robustness of media upload step                                                                |
| Example image generated: Batman typing on a laptop, used as default test prompt and slug                       | https://articles.emp0.com/wp-content/uploads/2025/07/img-batman-typing-on-a-laptop.jpg                     |
| Workflow supports external triggering with inputs, enabling integration in larger automation pipelines         | Via "When Executed by Another Workflow" node                                                             |

---

**Disclaimer:**  
The provided text is derived exclusively from an n8n automation workflow. It complies strictly with current content policies and contains no illegal, offensive, or protected material. All data handled is lawful and public.