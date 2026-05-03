Transform Images with AI-Enhanced Prompts using FLUX.1 Kontext and Mistral

https://n8nworkflows.xyz/workflows/transform-images-with-ai-enhanced-prompts-using-flux-1-kontext-and-mistral-5409


# Transform Images with AI-Enhanced Prompts using FLUX.1 Kontext and Mistral

### 1. Workflow Overview

This workflow automates AI-enhanced image transformation based on an initial base image and descriptive prompts stored in Airtable. Its main purpose is to generate visually rich, context-aware image prompts, transform base images using an AI image generation API (FLUX Kontext by Fal AI), verify the generation status, and update Airtable records with the final generated images.

The workflow is logically divided into these blocks:

- **1.1 Setup & Input Initialization**: Manual trigger and setup of API tokens.
- **1.2 Retrieve Base Data from Airtable**: Fetch base images and related descriptive fields.
- **1.3 AI Prompt Generation**: Use the Mistral Cloud Chat Model and Langchain to create an enhanced image generation prompt.
- **1.4 Image Generation via FLUX Kontext API**: Call the external API to generate images based on the prompt and base image.
- **1.5 Generation Verification & Retry Logic**: Check if the image was successfully generated; if not, wait and retry.
- **1.6 Log Generated Images back to Airtable**: Update Airtable records with the generated images.

---

### 2. Block-by-Block Analysis

#### 2.1 Setup & Input Initialization

- **Overview:**  
  This block initializes the workflow execution manually and sets up required API tokens for subsequent calls.

- **Nodes Involved:**  
  - When clicking ‘Execute workflow’  
  - Edit Fields

- **Node Details:**

  - **When clicking ‘Execute workflow’**  
    - Type: Manual Trigger  
    - Role: Entry point to execute the workflow manually.  
    - Configuration: No parameters needed.  
    - Inputs: None  
    - Outputs: Connects to “Edit Fields” node.  
    - Edge Cases: None expected; user-triggered.

  - **Edit Fields**  
    - Type: Set  
    - Role: Stores API token for Fal AI (FLUX Kontext) usage.  
    - Configuration: Sets a string variable `FalAITOKEN` with placeholder `[YOUR_API_TOKEN]`.  
    - Expressions: None; static assignment.  
    - Inputs: From manual trigger node.  
    - Outputs: Connects to “Airtable” node for data retrieval.  
    - Edge Cases: Missing or invalid API token will cause authentication errors downstream.

---

#### 2.2 Retrieve Base Data from Airtable

- **Overview:**  
  Fetches the base image URL, image description, situation, and name fields from an Airtable base and table. Filters for records where the `Prompt` field is empty.

- **Nodes Involved:**  
  - Airtable

- **Node Details:**

  - **Airtable**  
    - Type: Airtable node (v2.1)  
    - Role: Retrieves base image data and context for prompt generation.  
    - Configuration:  
      - Base: “YTB Outlier Finder”  
      - Table: “Image Generation”  
      - Operation: Search with filter formula `{Prompt} = ""` (empty prompt)  
      - Fields fetched: `ImageDescription`, `Situation`, `BaseImage`, `Name`  
    - Credentials: Airtable Personal Access Token (PAT)  
    - Inputs: From “Edit Fields”  
    - Outputs: Connects to “Generate Prompt”  
    - Edge Cases: No records found, API rate limits, or auth errors.

---

#### 2.3 AI Prompt Generation

- **Overview:**  
  Generates a detailed and imaginative image prompt by combining the base subject and situation using advanced language models.

- **Nodes Involved:**  
  - Mistral Cloud Chat Model  
  - Generate Prompt

- **Node Details:**

  - **Mistral Cloud Chat Model**  
    - Type: AI Language Model (Langchain node)  
    - Role: Provides the AI backend for generating text prompts.  
    - Configuration: Uses the “mistral-large-latest” model, no special options.  
    - Credentials: Mistral Cloud API key  
    - Inputs: (none directly; chained via Langchain)  
    - Outputs: Connects to “Generate Prompt” node as language model.  
    - Edge Cases: API quota limits, network issues, model errors.

  - **Generate Prompt**  
    - Type: Langchain Chain LLM (v1.7)  
    - Role: Defines the prompt engineering logic and prompt template for AI.  
    - Configuration:  
      - HumanMessagePromptTemplate with detailed instructions to create a vivid, scene-building prompt from two inputs: `ImageDescription` and `Situation` fields from Airtable.  
      - Output format: concise 1-3 sentence paragraph with visual keywords.  
    - Expressions:  
      - Uses `{{ $json.ImageDescription }}` and `{{ $json.Situation }}` from Airtable JSON.  
    - Inputs: From “Airtable” node (record data) and AI model node (Mistral)  
    - Outputs: Connects to “Generate Image” node.  
    - Edge Cases: Expression evaluation errors, empty inputs, AI model failures.

---

#### 2.4 Image Generation via FLUX Kontext API

- **Overview:**  
  Sends the AI-generated prompt and base image URL to the FLUX Kontext AI image generation API and requests an image transformation.

- **Nodes Involved:**  
  - Generate Image

- **Node Details:**

  - **Generate Image**  
    - Type: HTTP Request (v4.2)  
    - Role: Calls Fal AI’s FLUX Kontext “max” model API endpoint for image-to-image generation.  
    - Configuration:  
      - POST to `https://fal.run/fal-ai/flux-pro/kontext/max`  
      - Body parameters include:  
        - `prompt`: text from AI prompt generation (`{{ $json.text }}`)  
        - `image_url`: base image URL from Airtable (`{{ $('Airtable').item.json.BaseImage[0].url }}`)  
        - `guidance_scale`: 7.5 (controls adherence to prompt)  
        - `num_images`: 1  
        - `safety_tolerance`: 2  
        - `output_format`: jpeg  
        - `sync_mode`: false (async generation)  
        - `aspect_ratio`: 1:1  
      - Headers:  
        - `Authorization`: “Key {{ $('Edit Fields').item.json.FalAITOKEN }}”  
        - `Content-Type`: application/json  
      - Retry on failure enabled  
    - Inputs: From “Generate Prompt” node (prompt text)  
    - Outputs: Connects to “Check IF Generated” node and also direct error output (empty)  
    - Edge Cases: API auth failures, request timeouts, malformed responses, rate limiting.

---

#### 2.5 Generation Verification & Retry Logic

- **Overview:**  
  Verifies if the image has been generated successfully by checking the returned image URL. If not ready, waits 3 seconds and retries the check.

- **Nodes Involved:**  
  - Check IF Generated  
  - Wait

- **Node Details:**

  - **Check IF Generated**  
    - Type: HTTP Request (v4.2)  
    - Role: Sends a GET request to the URL of the generated image to verify availability.  
    - Configuration:  
      - URL: dynamically set to `{{ $json.images[0].url }}` (image URL from previous node)  
      - On error: continue workflow with error output (non-blocking)  
    - Inputs: From “Generate Image” node  
    - Outputs:  
      - On success: connects to “Log Image Posts” node  
      - On error: connects to “Wait” node for retry  
    - Edge Cases: 404 Not Found if image not available yet, network errors, timeout.

  - **Wait**  
    - Type: Wait (v1.1)  
    - Role: Pauses execution for 3 seconds before retrying the image availability check.  
    - Configuration: Wait for 3 seconds.  
    - Inputs: From error branch of “Check IF Generated” node  
    - Outputs: Connects back to “Check IF Generated” node (loop for retry)  
    - Edge Cases: Infinite loop if image never becomes available (no max retry count).

---

#### 2.6 Log Generated Images back to Airtable

- **Overview:**  
  Updates the Airtable record with the generated prompt and final image URL to log the AI-transformed image.

- **Nodes Involved:**  
  - Log Image Posts

- **Node Details:**

  - **Log Image Posts**  
    - Type: Airtable node (v2.1)  
    - Role: Updates the Airtable table “Table 2” with the transformed image and prompt information.  
    - Configuration:  
      - Base: “YTB Outlier Finder”  
      - Table: “Table 2”  
      - Operation: Update record matched by “Name” field  
      - Columns mapped:  
        - `Name`: from Airtable original record `Name`  
        - `Prompt`: from generated prompt text  
        - `FinalImage`: array with URL of generated image (`[{ "url": $json.images[0].url }]`)  
    - Credentials: Airtable Personal Access Token  
    - Inputs: From successful output of “Check IF Generated” node  
    - Outputs: None (terminal node)  
    - Edge Cases: Airtable API errors, record not found, permission issues.

---

### 3. Summary Table

| Node Name                 | Node Type                          | Functional Role                               | Input Node(s)                  | Output Node(s)              | Sticky Note                                                                                 |
|---------------------------|----------------------------------|-----------------------------------------------|-------------------------------|-----------------------------|---------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger                  | Workflow manual start                         | None                          | Edit Fields                 | ## Base Image & Prompt                                                                       |
| Edit Fields               | Set                              | Sets API tokens                               | When clicking ‘Execute workflow’ | Airtable                   | ## Base Image & Prompt                                                                       |
| Airtable                  | Airtable (v2.1)                  | Retrieve base image and prompt data           | Edit Fields                   | Generate Prompt             | ## Base Image & Prompt                                                                       |
| Mistral Cloud Chat Model  | AI Language Model (Langchain)    | Provides AI model for prompt generation       | None (AI model)               | Generate Prompt (ai_languageModel) |                                                                                             |
| Generate Prompt           | Langchain Chain LLM (v1.7)       | Generate enhanced image prompt                 | Airtable, Mistral Cloud Chat Model | Generate Image             |                                                                                             |
| Generate Image            | HTTP Request (v4.2)              | Call image generation API                      | Generate Prompt               | Check IF Generated          | ## Image Generation                                                                         |
| Check IF Generated        | HTTP Request (v4.2)              | Verify image generation status                 | Generate Image                | Log Image Posts, Wait       | ## Image Generation                                                                         |
| Wait                      | Wait (v1.1)                     | Wait before retrying image availability check | Check IF Generated (error)    | Check IF Generated          | ## Image Generation                                                                         |
| Log Image Posts           | Airtable (v2.1)                  | Update Airtable with generated image and prompt | Check IF Generated (success)  | None                       | ## Image Generation                                                                         |
| Sticky Note               | Sticky Note                     | Annotation: Image Generation section           | None                         | None                       | ## Image Generation                                                                         |
| Sticky Note1              | Sticky Note                     | Annotation: Base Image & Prompt section        | None                         | None                       | ## Base Image & Prompt                                                                       |
| Sticky Note2              | Sticky Note                     | Example base image and prompt illustration     | None                         | None                       | ## Base Image                                                                               |
| Sticky Note3              | Sticky Note                     | Example generated image illustration           | None                         | None                       | ## Generated Image                                                                          |
| Sticky Note5              | Sticky Note                     | Explanation of workflow steps and YouTube link | None                         | None                       | ## How it works? See video: https://www.youtube.com/watch?v=0SVj70-dA0Q                     |
| Sticky Note6              | Sticky Note                     | Setup instructions and API references          | None                         | None                       | ## SETUP instructions and API links                                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: `When clicking ‘Execute workflow’`  
   - Type: Manual Trigger  
   - No extra configuration.

2. **Create Set Node to Store API Token**  
   - Name: `Edit Fields`  
   - Type: Set  
   - Add a string field named `FalAITOKEN` with value `[YOUR_API_TOKEN]` (replace with actual Fal AI token).  
   - Connect `When clicking ‘Execute workflow’` → `Edit Fields`.

3. **Create Airtable Node to Fetch Base Data**  
   - Name: `Airtable`  
   - Type: Airtable (v2.1)  
   - Credentials: Connect Airtable Personal Access Token (PAT)  
   - Base: Select “YTB Outlier Finder”  
   - Table: Select “Image Generation”  
   - Operation: Search  
   - Filter formula: `{Prompt} = ""` (to get records missing a prompt)  
   - Fields: `ImageDescription`, `Situation`, `BaseImage`, `Name`  
   - Connect `Edit Fields` → `Airtable`.

4. **Create Mistral Cloud Chat Model Node**  
   - Name: `Mistral Cloud Chat Model`  
   - Type: AI Language Model (Langchain)  
   - Credentials: Mistral Cloud API key  
   - Model: `mistral-large-latest`  
   - No additional options.

5. **Create Langchain Chain LLM Node to Generate Prompt**  
   - Name: `Generate Prompt`  
   - Type: Langchain Chain LLM (v1.7)  
   - Prompt template: Use HumanMessagePromptTemplate with instructions to generate imaginative image prompt from `ImageDescription` and `Situation`.  
   - Bind inputs: Use `{{ $json.ImageDescription }}` and `{{ $json.Situation }}` from Airtable data.  
   - Connect `Airtable` → `Generate Prompt` (main input), and connect `Mistral Cloud Chat Model` → `Generate Prompt` (ai_languageModel input).

6. **Create HTTP Request Node to Call Image Generation API**  
   - Name: `Generate Image`  
   - Type: HTTP Request (v4.2)  
   - Method: POST  
   - URL: `https://fal.run/fal-ai/flux-pro/kontext/max`  
   - Headers:  
     - `Authorization`: `Key {{ $('Edit Fields').item.json.FalAITOKEN }}`  
     - `Content-Type`: `application/json`  
   - Body Parameters (JSON):  
     - `prompt`: `={{ $json.text }}` (from Generate Prompt output)  
     - `image_url`: `={{ $('Airtable').item.json.BaseImage[0].url }}`  
     - `guidance_scale`: 7.5  
     - `num_images`: 1  
     - `safety_tolerance`: 2  
     - `output_format`: jpeg  
     - `sync_mode`: false  
     - `aspect_ratio`: 1:1  
   - Enable retry on failure.  
   - Connect `Generate Prompt` → `Generate Image`.

7. **Create HTTP Request Node to Check Image Generation**  
   - Name: `Check IF Generated`  
   - Type: HTTP Request (v4.2)  
   - Method: GET (default)  
   - URL: `={{ $json.images[0].url }}` (from Generate Image output)  
   - On Error: Continue workflow on error output.  
   - Connect `Generate Image` → `Check IF Generated`.

8. **Create Wait Node to Retry**  
   - Name: `Wait`  
   - Type: Wait (v1.1)  
   - Configure to wait 3 seconds.  
   - Connect error output of `Check IF Generated` → `Wait`.  
   - Connect `Wait` → `Check IF Generated` (loop to retry).

9. **Create Airtable Node to Log Generated Image**  
   - Name: `Log Image Posts`  
   - Type: Airtable (v2.1)  
   - Credentials: Same Airtable PAT as before  
   - Base: “YTB Outlier Finder”  
   - Table: “Table 2”  
   - Operation: Update record matching by `Name` field  
   - Columns mapping:  
     - `Name`: `={{ $('Airtable').item.json.Name }}`  
     - `Prompt`: `={{ $json.prompt }}` (generated prompt)  
     - `FinalImage`: `={{ [{ "url": $json.images[0].url }] }}`  
   - Connect success output of `Check IF Generated` → `Log Image Posts`.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                             | Context or Link                                                                                         |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| Setup input can be replaced with any source providing a prompt and a publicly accessible base image URL. The output currently stores generated images back to Airtable but can be modified to store elsewhere or use images directly in n8n by switching `sync_mode` to true and extracting image data as Base64.                                                                                 | Setup Sticky Note                                                                                      |
| API integrations require replacing `[YOUR_API_TOKEN]` with valid tokens or connecting via OAuth2/Client ID & Secret as supported in n8n.                                                                                                                                                                                                                                               | Setup Sticky Note                                                                                      |
| Workflow steps summarized: 1) Retrieve base image and description from Airtable; 2) Generate enhanced prompt via Mistral AI; 3) Generate image using Fal AI FLUX Kontext; 4) Poll and verify image generation; 5) Log results back to Airtable.                                                                                                                                              | Workflow Explanation Sticky Note with YouTube video: https://www.youtube.com/watch?v=0SVj70-dA0Q     |
| Example base image and prompt (visual aid) and example generated image are included in sticky notes for reference.                                                                                                                                                                                                                                                                         | Sticky Notes with images                                                                                |
| FLUX Kontext API documentation for image generation: https://fal.ai/models/fal-ai/flux-pro/kontext/max/api#schema-input                                                                                                                                                                                                                                                                   | API Reference Link                                                                                    |
| Airtable node documentation: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.airtable/?utm_source=n8n_app&utm_medium=node_settings_modal-credential_link&utm_campaign=n8n-nodes-base.airtable                                                                                                                                    | Airtable Node Docs                                                                                     |

---

**Disclaimer:**  
The provided text is exclusively derived from an automated workflow created using n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled are legal and public.