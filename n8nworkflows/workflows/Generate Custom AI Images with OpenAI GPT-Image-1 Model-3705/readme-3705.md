Generate Custom AI Images with OpenAI GPT-Image-1 Model

https://n8nworkflows.xyz/workflows/generate-custom-ai-images-with-openai-gpt-image-1-model-3705


# Generate Custom AI Images with OpenAI GPT-Image-1 Model

### 1. Workflow Overview

This workflow enables users to generate custom AI images using OpenAI’s GPT-Image-1 model via the OpenAI image generation API. It is designed for manual triggering through the n8n UI and supports generating multiple images per request with configurable parameters such as prompt text, image size, quality, and model selection.

The workflow is logically divided into the following blocks:

- **1.1 Input Initialization:** Manual trigger and setting of key parameters for image generation.
- **1.2 Image Generation API Call:** Sending a POST request to OpenAI’s image generation endpoint with the configured parameters.
- **1.3 Response Handling:** Splitting the API response to process multiple images individually.
- **1.4 Image Conversion:** Converting base64-encoded image data into downloadable binary files for further use or storage.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Initialization

- **Overview:**  
  This block initiates the workflow manually and defines all necessary input parameters for the image generation process, such as the prompt, number of images, image size, quality, and model.

- **Nodes Involved:**  
  - When clicking ‘Test workflow’ (Manual Trigger)  
  - Set Variables

- **Node Details:**

  - **When clicking ‘Test workflow’**  
    - Type: Manual Trigger  
    - Role: Entry point to start the workflow manually via n8n UI.  
    - Configuration: No parameters; simply triggers the workflow on user action.  
    - Inputs: None  
    - Outputs: Connects to "Set Variables" node.  
    - Edge Cases: None specific; user must trigger manually.

  - **Set Variables**  
    - Type: Set  
    - Role: Defines and assigns key variables used in the API request.  
    - Configuration:  
      - `image_prompt`: "a 4-frame cartoon strip telling a joke about AI"  
      - `number_of_images`: 2  
      - `quality_of_image`: "high"  
      - `size_of_image`: "1024x1024"  
      - `openai_image_model`: "gpt-image-1" (expression used: `=gpt-image-1`)  
    - Inputs: Receives trigger from Manual Trigger node.  
    - Outputs: Passes variables to the HTTP Request node for image generation.  
    - Edge Cases: Incorrect variable types or missing values could cause API errors.

#### 1.2 Image Generation API Call

- **Overview:**  
  This block sends a POST request to OpenAI’s image generation API endpoint with the parameters set previously, requesting image generation.

- **Nodes Involved:**  
  - OpenAI - Generate Image (HTTP Request)

- **Node Details:**

  - **OpenAI - Generate Image**  
    - Type: HTTP Request  
    - Role: Calls OpenAI’s image generation API to create images based on the prompt and parameters.  
    - Configuration:  
      - URL: `https://api.openai.com/v1/images/generations`  
      - Method: POST  
      - Headers: Content-Type: application/json  
      - Authentication: Uses predefined OpenAI API credentials (`openAiApi`)  
      - Body (JSON): Dynamically constructed using expressions referencing variables:  
        ```json
        {
          "model": "{{ $json.openai_image_model }}",
          "prompt": "{{ $json.image_prompt }}",
          "n": {{ $json.number_of_images }},
          "size": "{{ $json.size_of_image }}",
          "quality": "{{ $json.quality_of_image }}"
        }
        ```  
    - Inputs: Receives variables from "Set Variables" node.  
    - Outputs: API response JSON containing generated images data.  
    - Version Requirements: Requires n8n version supporting HTTP Request node v4.2 or higher for advanced JSON body expressions.  
    - Edge Cases:  
      - Authentication errors if API key is invalid or missing.  
      - API rate limits or quota exceeded errors.  
      - Verification requirement errors if OpenAI organization verification is incomplete (see Important Note).  
      - Malformed JSON or missing parameters causing API rejection.

#### 1.3 Response Handling

- **Overview:**  
  This block processes the API response by splitting the array of generated images so each image can be handled individually.

- **Nodes Involved:**  
  - Separate Image Outputs (Split Out)

- **Node Details:**

  - **Separate Image Outputs**  
    - Type: Split Out  
    - Role: Splits the `data` array from the API response into individual items for separate processing.  
    - Configuration: Field to split out: `data` (the array containing image objects).  
    - Inputs: Receives API response from "OpenAI - Generate Image".  
    - Outputs: Each output item contains one image object with base64 data.  
    - Edge Cases:  
      - Empty or missing `data` field in response could cause no outputs.  
      - Unexpected response structure may cause failure.

#### 1.4 Image Conversion

- **Overview:**  
  Converts each base64-encoded image string into a binary file format suitable for download or further processing.

- **Nodes Involved:**  
  - Convert to File

- **Node Details:**

  - **Convert to File**  
    - Type: Convert To File  
    - Role: Converts base64 image data (`b64_json`) into binary file format.  
    - Configuration:  
      - Operation: toBinary  
      - Source Property: `b64_json` (the base64 string from each image object)  
    - Inputs: Receives split image data from "Separate Image Outputs".  
    - Outputs: Binary files representing each generated image.  
    - Edge Cases:  
      - Invalid or corrupted base64 data could cause conversion failure.  
      - Missing `b64_json` property in input data.

---

### 3. Summary Table

| Node Name               | Node Type           | Functional Role                      | Input Node(s)            | Output Node(s)          | Sticky Note                                                                                                                  |
|-------------------------|---------------------|------------------------------------|--------------------------|-------------------------|------------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger      | Starts workflow manually            | None                     | Set Variables           |                                                                                                                              |
| Set Variables           | Set                 | Defines image generation parameters | When clicking ‘Test workflow’ | OpenAI - Generate Image |                                                                                                                              |
| OpenAI - Generate Image | HTTP Request        | Calls OpenAI image generation API   | Set Variables            | Separate Image Outputs   |                                                                                                                              |
| Separate Image Outputs  | Split Out           | Splits API response array into items | OpenAI - Generate Image  | Convert to File          |                                                                                                                              |
| Convert to File         | Convert To File     | Converts base64 image data to binary | Separate Image Outputs   | None                    |                                                                                                                              |
| Sticky Note             | Sticky Note         | Provides video tutorial and API links | None                     | None                    | ## [CLICK HERE to Watch Video](https://youtu.be/YmDezgolqzU?si=BgMjRm55-T_CYAs7) OpenAI just dropped API access for their new image generation — and it changes everything. In this quick walkthrough, I show you exactly how to integrate it with n8n using an HTTP request node. Learn how to send prompts, convert base64 to binary, and automate image handling. This is a big one. Don’t miss it. 🔗 Official API Overview: https://openai.com/index/image-generation-api/ 🔗 API Reference – Create Image: https://platform.openai.com/docs/api-reference/images/create ### *New:  Make.com scenario here: https://drive.google.com/file/d/1Uz-mA0LnUZ_tnUWBR2AAlVxs3LBlGKfk/view?usp=sharing |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: `When clicking ‘Test workflow’`  
   - Type: Manual Trigger  
   - No parameters needed. This node will start the workflow manually.

2. **Create Set Node**  
   - Name: `Set Variables`  
   - Type: Set  
   - Connect input from `When clicking ‘Test workflow’` node.  
   - Add the following variables with exact types and values:  
     - `image_prompt` (string): `"a 4-frame cartoon strip telling a joke about AI"`  
     - `number_of_images` (number): `2`  
     - `quality_of_image` (string): `"high"`  
     - `size_of_image` (string): `"1024x1024"`  
     - `openai_image_model` (string): Use expression `=gpt-image-1` (or simply set as string `"gpt-image-1"`)

3. **Create HTTP Request Node**  
   - Name: `OpenAI - Generate Image`  
   - Type: HTTP Request  
   - Connect input from `Set Variables` node.  
   - Configure:  
     - URL: `https://api.openai.com/v1/images/generations`  
     - Method: POST  
     - Authentication: Select or create OpenAI API credentials (OAuth2 or API key) named `openAiApi` or similar.  
     - Headers: Add header `Content-Type: application/json`  
     - Body Content Type: JSON  
     - Body Parameters: Use raw JSON with expressions referencing incoming JSON variables:  
       ```json
       {
         "model": "{{ $json.openai_image_model }}",
         "prompt": "{{ $json.image_prompt }}",
         "n": {{ $json.number_of_images }},
         "size": "{{ $json.size_of_image }}",
         "quality": "{{ $json.quality_of_image }}"
       }
       ```  
     - Enable sending body and headers.

4. **Create Split Out Node**  
   - Name: `Separate Image Outputs`  
   - Type: Split Out  
   - Connect input from `OpenAI - Generate Image` node.  
   - Configure:  
     - Field to split out: `data` (this is the array in the API response containing images)

5. **Create Convert To File Node**  
   - Name: `Convert to File`  
   - Type: Convert To File  
   - Connect input from `Separate Image Outputs` node.  
   - Configure:  
     - Operation: `toBinary`  
     - Source Property: `b64_json` (the base64 image data property from each split item)

6. **Optional: Add Sticky Note**  
   - Add a Sticky Note node anywhere on the canvas with the following content for user reference:  
     ```
     ## [CLICK HERE to Watch Video](https://youtu.be/YmDezgolqzU?si=BgMjRm55-T_CYAs7)

     OpenAI just dropped API access for their new image generation — and it changes everything. In this quick walkthrough, I show you exactly how to integrate it with n8n using an HTTP request node. Learn how to send prompts, convert base64 to binary, and automate image handling. This is a big one. Don’t miss it.

     🔗 Official API Overview: https://openai.com/index/image-generation-api/
     🔗 API Reference – Create Image: https://platform.openai.com/docs/api-reference/images/create

     ### *New:  Make.com scenario here: https://drive.google.com/file/d/1Uz-mA0LnUZ_tnUWBR2AAlVxs3LBlGKfk/view?usp=sharing
     ```

7. **Save and Activate Workflow**  
   - Ensure the OpenAI API credentials are valid and that your OpenAI organization verification is complete as per [OpenAI's verification requirements](https://help.openai.com/en/articles/10910291-api-organization-verification).  
   - Test the workflow by manually triggering it and verify images are generated and converted properly.

---

### 5. General Notes & Resources

| Note Content                                                                                                                             | Context or Link                                                                                                                  |
|------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------|
| You must complete OpenAI's new organization verification to use their image generation API. It is free and quick to do.                  | https://help.openai.com/en/articles/10910291-api-organization-verification                                                      |
| Official OpenAI Image Generation API overview and documentation.                                                                          | https://openai.com/index/image-generation-api/                                                                                   |
| OpenAI API Reference for the Create Image endpoint.                                                                                       | https://platform.openai.com/docs/api-reference/images/create                                                                     |
| Video walkthrough demonstrating integration of OpenAI image generation with n8n, including prompt sending and base64 to binary conversion. | https://youtu.be/YmDezgolqzU?si=BgMjRm55-T_CYAs7                                                                                 |
| Additional Make.com scenario related to this workflow for extended automation examples.                                                   | https://drive.google.com/file/d/1Uz-mA0LnUZ_tnUWBR2AAlVxs3LBlGKfk/view?usp=sharing                                               |

---

This documentation provides a complete, structured understanding of the workflow, enabling users and automation agents to reproduce, modify, and troubleshoot the image generation process using OpenAI’s GPT-Image-1 model within n8n.