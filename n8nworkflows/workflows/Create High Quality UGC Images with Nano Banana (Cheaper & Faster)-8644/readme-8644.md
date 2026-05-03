Create High Quality UGC Images with Nano Banana (Cheaper & Faster)

https://n8nworkflows.xyz/workflows/create-high-quality-ugc-images-with-nano-banana--cheaper---faster--8644


# Create High Quality UGC Images with Nano Banana (Cheaper & Faster)

### 1. Workflow Overview

This workflow automates the creation of high-quality User-Generated Content (UGC) style marketing images using AI, specifically leveraging Fal.ai’s Nano Banana image generation model. It targets content creators, e-commerce stores, and marketing teams who want to quickly generate diverse, realistic, and visually appealing UGC images of their products with minimal manual effort.

The workflow operates in several logical blocks:

- **1.1 Input Reception**: Watches a specific Google Drive folder for newly uploaded product images (ideally with white background).
- **1.2 Setup & Configuration**: Reads API keys and sets global parameters necessary for downstream nodes.
- **1.3 UGC Prompt Generation**: Uses an advanced AI language model (Mistral Cloud) to generate 50 diverse and creative image prompts describing various scenarios featuring the product.
- **1.4 Prompt Handling & Looping**: Processes the generated prompts individually in a batch loop.
- **1.5 Image Generation**: For each prompt, calls Fal.ai’s image generation API (Nano Banana) to create a UGC image based on the prompt and the base product image.
- **1.6 Image Retrieval & Upload**: Retrieves the generated image and uploads it back to the same Google Drive folder under a systematically named file.
- **1.7 No Operation Step**: Placeholder or optional node to handle flow control after image upload.

Supplemental sticky notes provide setup instructions, workflow explanation, and example images.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:** Detects new product image files uploaded to a specific Google Drive folder named "UGC" to trigger the workflow.
- **Nodes Involved:** `Google Drive Trigger`
- **Node Details:**
  - Type: Google Drive Trigger node
  - Configuration: Watches for `fileCreated` event, polling every minute, restricted to a specified folder called "UGC".
  - Inputs: N/A (trigger node)
  - Outputs: Emits details of newly uploaded files including file ID and name.
  - Edge cases: Permissions issues if the folder or files are not publicly accessible, API quota limits.
  - Credentials: Requires Google Drive OAuth2 API credentials.
  - Notes: The trigger folder and its contents must be publicly accessible for downstream image generation.

#### 1.2 Setup & Configuration

- **Overview:** Sets necessary API keys and environment parameters used by other nodes.
- **Nodes Involved:** `Setup`
- **Node Details:**
  - Type: Set node
  - Configuration: Defines a string variable `FalAPIKey` with the Fal.ai API key placeholder `[YOUR_API_KEY]`.
  - Inputs: Receives file metadata from Google Drive Trigger.
  - Outputs: Passes along with the new variable for API authentication.
  - Edge cases: Missing or incorrect API key will cause failures in HTTP request nodes.
  - Notes: User must manually replace placeholder with a valid API key.

#### 1.3 UGC Prompt Generation

- **Overview:** Generates 50 unique, diverse, and realistic text prompts describing scenarios where the product is used, to guide AI image generation.
- **Nodes Involved:** `Mistral Cloud Chat Model`, `Structured Output Parser`, `Generate Prompts`, `Split Out`
- **Node Details:**

  - **Mistral Cloud Chat Model:**
    - Type: AI Language Model node (Mistral Cloud)
    - Configuration: Uses `mistral-large-latest` model, no special options.
    - Inputs: Receives trigger data and API setup.
    - Outputs: Raw AI chat completion.
    - Credentials: Mistral Cloud API credentials.
    - Edge cases: API rate limits, malformed prompts, network timeouts.

  - **Structured Output Parser:**
    - Type: LangChain Structured Output Parser
    - Configuration: Parses AI's JSON response with schema expecting an array of `ugc_prompts` each with `prompt_number` and `prompt` string.
    - Inputs: Receives raw AI output.
    - Outputs: Parsed JSON array of prompts.
    - Edge cases: Parsing errors if AI output deviates from expected schema.

  - **Generate Prompts:**
    - Type: Chain LLM node
    - Configuration: Sends a detailed prompt instructing the model to generate 50 distinct image scenario prompts for the product identified by the file name (stripped of extension).
    - Inputs: Receives initial setup and trigger data.
    - Outputs: AI-generated prompt JSON.
    - Edge cases: Incorrect product name extraction may affect prompt relevance.

  - **Split Out:**
    - Type: Split Out node
    - Configuration: Splits the array of prompts into individual items for looping.
    - Inputs: Receives parsed prompt array.
    - Outputs: Single prompt per execution.
    - Edge cases: Empty or malformed prompt arrays.

#### 1.4 Prompt Handling & Looping

- **Overview:** Loops over each individual prompt to process image generation and upload.
- **Nodes Involved:** `Loop Over Items`, `No Operation, do nothing`
- **Node Details:**
  - Type: Split In Batches node
  - Configuration: Processes one prompt per batch.
  - Inputs: Individual prompts from `Split Out`.
  - Outputs: Feeds each prompt separately into the image generation branch.
  - Edge cases: Failures in looping logic may cause skipped prompts.
  - No Operation node:
    - Serves as a flow control or placeholder after loop.
    - No inputs or outputs except connection from loop.
    - Edge cases: None functional.

#### 1.5 Image Generation

- **Overview:** Sends each prompt along with the base product image URL to Fal.ai’s Nano Banana image generation API, requesting one UGC image per prompt.
- **Nodes Involved:** `Generate Image`
- **Node Details:**
  - Type: HTTP Request node
  - Configuration:
    - POST request to `https://fal.run/fal-ai/gemini-25-flash-image/edit`
    - JSON body includes:
      - `prompt`: current prompt string
      - `image_urls`: array containing Google Drive public URL constructed from trigger file ID
      - `num_images`: 1
      - `output_format`: "jpeg"
      - `sync_mode`: false (asynchronous)
    - Headers include Authorization with FalAPIKey and Content-Type JSON.
  - Inputs: Prompt from loop and API key from setup.
  - Outputs: API response with generated image metadata.
  - Error handling: Continue on failure to avoid full workflow halt.
  - Edge cases: API rate limits, malformed requests, invalid URLs, network errors.
  - Notes: Requires base image to be publicly accessible on Google Drive.

#### 1.6 Image Retrieval & Upload

- **Overview:** Retrieves the generated image from the URL provided by Fal.ai and uploads it back to the designated Google Drive folder with a uniquely constructed file name.
- **Nodes Involved:** `Retrieve Image`, `Upload file`
- **Node Details:**
  - **Retrieve Image:**
    - Type: HTTP Request node
    - Configuration: GET request to the first image URL in the Fal.ai response.
    - Inputs: Receives generated image URL from `Generate Image`.
    - Outputs: Binary image data.
    - Edge cases: URL not reachable, image not generated yet if async delay occurs.
  
  - **Upload file:**
    - Type: Google Drive node (upload file)
    - Configuration:
      - File name constructed by concatenating original product name (without extension) and the prompt number.
      - Upload location is the same Google Drive folder as the trigger.
    - Inputs: Binary image data from `Retrieve Image`.
    - Outputs: Confirmation of upload.
    - Credentials: Google Drive OAuth2 API.
    - Edge cases: Upload permission errors, name conflicts.

---

### 3. Summary Table

| Node Name                 | Node Type                                    | Functional Role                          | Input Node(s)             | Output Node(s)           | Sticky Note                                                  |
|---------------------------|----------------------------------------------|----------------------------------------|---------------------------|--------------------------|--------------------------------------------------------------|
| Google Drive Trigger       | Google Drive Trigger                          | Detect new product image upload        | N/A                       | Setup                    | Must have publicly accessible folder/files                   |
| Setup                     | Set                                           | Define API key and environment variables | Google Drive Trigger      | Generate Prompts         | Replace placeholder with actual Fal.ai API key               |
| Mistral Cloud Chat Model  | AI Language Model (LangChain)                 | Generate raw UGC prompts JSON          | Setup                     | Structured Output Parser |                                                              |
| Structured Output Parser  | LangChain Structured Output Parser            | Parse AI JSON output to structured data | Mistral Cloud Chat Model  | Generate Prompts         |                                                              |
| Generate Prompts          | Chain LLM Node                               | Send prompt to AI for UGC prompt generation | Setup                     | Split Out                | Contains detailed prompt instructions, customizable prompt count |
| Split Out                 | Split Out                                    | Split prompt array to individual prompts | Generate Prompts          | Loop Over Items          |                                                              |
| Loop Over Items           | Split In Batches                             | Iterate over each prompt                | Split Out                 | No Operation, Generate Image |                                                              |
| No Operation, do nothing  | No Operation                                 | Flow control placeholder                | Loop Over Items           |                          |                                                              |
| Generate Image            | HTTP Request                                | Call Fal.ai Nano Banana image generator | Loop Over Items           | Retrieve Image           | API key needed; continues on error                           |
| Retrieve Image            | HTTP Request                                | Download generated image binary         | Generate Image            | Upload file              |                                                              |
| Upload file               | Google Drive                                | Upload generated image back to Drive    | Retrieve Image            | Loop Over Items          | Output file named by product name + prompt number           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Google Drive Trigger node:**
   - Set event to `fileCreated`.
   - Set polling interval to every minute.
   - Configure folder to watch to the "UGC" folder in your Google Drive.
   - Authenticate with Google Drive OAuth2 credentials.
   - Ensure the folder and its contents are publicly accessible.

2. **Add a Set node named `Setup`:**
   - Add a string field `FalAPIKey`.
   - Set value to your Fal.ai API key (replace `[YOUR_API_KEY]`).
   - Connect Google Drive Trigger output to this node.

3. **Add Mistral Cloud Chat Model node:**
   - Select model: `mistral-large-latest`.
   - Authenticate with your Mistral Cloud API credentials.
   - Connect from `Setup`.
   - Configure prompt message:
     ```
     The product the Promote is a {{ $('Google Drive Trigger').item.json.name.match(/^[^.]+/)[0] }}

     Your task is to generate 50 creative, diverse, and realistic situations prompts for an image-to-image generation model. Each prompt should describe a different situation or scenario where the product is being used.

     Requirements for each prompt:
     1. Specify the environment or setting (e.g., home, office, kitchen).
     2. Include a human or realistic context interacting with the product (e.g., holding it, using it, displaying it).
     3. Highlight at least one functionality or feature of the product in the scene.
     4. Keep the prompts concise (1–2 sentences), vivid, and visually descriptive.
     5. Maintain diversity: do not repeat the same type of scene. Include different times of day, moods, activities, or locations.

     Output format (JSON array):
     {
           "ugc_prompts": [
                   {"prompt": "First prompt here..."},
                   {"prompt": "Second prompt here..."},
                   ...
                   {"prompt": "Fiftieth prompt here..."}
                           ]
     }

     DON'T RETURN ANYTHING ELSE THAN THE JSON!
     ```

4. **Add LangChain Structured Output Parser node:**
   - Configure with JSON schema expecting an array of `ugc_prompts` each with `prompt_number` (integer) and `prompt` (string).
   - Connect from Mistral Cloud Chat Model node’s AI output.

5. **Add Chain LLM node named `Generate Prompts`:**
   - Configure to use the Structured Output Parser.
   - Connect from Setup node to Mistral Cloud node and then to Structured Output Parser (per output parser input).
   - Connect Structured Output Parser output to `Generate Prompts`.

6. **Add Split Out node:**
   - Configure to split the field `output.ugc_prompts`.
   - Connect from `Generate Prompts` node.

7. **Add Split In Batches node named `Loop Over Items`:**
   - No special batch size needed (default is 1).
   - Connect from `Split Out`.

8. **Add HTTP Request node named `Generate Image`:**
   - POST to `https://fal.run/fal-ai/gemini-25-flash-image/edit`.
   - Set Headers:
     - `Authorization`: `Key {{ $('Setup').first().json.FalAPIKey }}`
     - `Content-Type`: `application/json`
   - JSON Body:
     ```json
     {
       "prompt": "{{ $json.prompt }}",
       "image_urls": [
         "https://drive.google.com/uc?export=view&id={{ $('Google Drive Trigger').first().json.id }}"
       ],
       "num_images": 1,
       "output_format": "jpeg",
       "sync_mode": false
     }
     ```
   - On error: continue regular output.
   - Connect from `Loop Over Items`.

9. **Add HTTP Request node named `Retrieve Image`:**
   - GET request to URL: `={{ $json.images[0].url }}`
   - Connect from `Generate Image`.

10. **Add Google Drive node named `Upload file`:**
    - Set file name: `={{ $('Google Drive Trigger').first().json.name.match(/^[^.]+/)[0].concat($('Loop Over Items').item.json.prompt_number) }}`
    - Select Drive: "My Drive".
    - Set folder to the same "UGC" folder.
    - Connect from `Retrieve Image`.
    - Authenticate with Google Drive OAuth2 API.

11. **Add No Operation node named `No Operation, do nothing`:**
    - Connect from `Loop Over Items` (main output).
    - Optional placeholder or for future extensions.

12. **Connect `Upload file` node output back to `Loop Over Items` (second output) to continue looping.**

---

### 5. General Notes & Resources

| Note Content                                                                                                             | Context or Link                                                                                         |
|--------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| Workflow requires product base images with preferably white backgrounds for best image generation results.               | Setup instructions                                                                                     |
| Google Drive folder and contained files must be publicly accessible to allow Fal.ai API to fetch base images.            | Setup instructions                                                                                     |
| Customizable parameters include: number of generated prompts, base image URLs, number of images per prompt, and upload folder. | Sticky Note6                                                                                           |
| Workflow cost estimate: approximately $0.039 per generated image using Fal.ai Nano Banana engine.                        | Sticky Note9                                                                                           |
| YouTube video tutorial available showcasing workflow usage and setup: [Watch on YouTube](https://www.youtube.com/watch?v=0SVj70-dA0Q) | Sticky Note5                                                                                           |
| Example base and output images available for reference in Sticky Note7.                                                  | Sticky Note7                                                                                           |
| Google Drive OAuth2 API setup details: https://docs.n8n.io/integrations/builtin/credentials/google/                      | Setup instructions                                                                                     |
| Fal.ai API documentation for image generation model: https://fal.ai/models/fal-ai/gemini-25-flash-image/edit/api          | Setup instructions                                                                                     |

---

**Disclaimer:**  
The provided description is based exclusively on an automated workflow created in n8n, an integration and automation platform. The processing strictly adheres to content policies, containing no illegal, offensive, or protected elements. All processed data is legal and public.