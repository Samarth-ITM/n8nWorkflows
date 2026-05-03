Generate Images with OpenAI new GPT-Image-1 Model via User-Friendly Form

https://n8nworkflows.xyz/workflows/generate-images-with-openai-new-gpt-image-1-model-via-user-friendly-form-3688


# Generate Images with OpenAI new GPT-Image-1 Model via User-Friendly Form

### 1. Workflow Overview

This workflow, titled **Simple OpenAI Image Generator**, enables users to generate images using OpenAI’s new GPT-Image-1 model through a user-friendly web form. It is designed for scenarios where users want to create AI-generated images by providing a textual prompt and selecting an image size.

The workflow is logically divided into three main blocks:

- **1.1 Input Reception:** Captures user input from a web form, including the image prompt and desired image size.
- **1.2 AI Image Generation:** Sends the user input to the OpenAI API to generate an image based on the GPT-Image-1 model.
- **1.3 Output Delivery:** Converts the generated image data into a downloadable file format and returns it to the user via the form interface.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block collects user input through a web form. Users provide an image prompt and select the desired image size from predefined options.

- **Nodes Involved:**  
  - Prompt and options

- **Node Details:**

  - **Prompt and options**  
    - *Type and Role:* Form Trigger node; initiates the workflow upon form submission.  
    - *Configuration:*  
      - Form title: "OpenAI Image Generator"  
      - Two form fields:  
        - **Prompt:** Text input, required, placeholder "Snow-covered mountain village in the Alps"  
        - **Image size:** Dropdown with options "1024x1024", "1024x1536", "1536x1024", required  
      - Webhook ID assigned for external access.  
    - *Key Expressions/Variables:*  
      - `$json.Prompt` for the image prompt input  
      - `$json['Image size']` for the selected image size  
    - *Input/Output Connections:*  
      - No input connections (trigger node)  
      - Output connected to "OpenAI Image Generation" node  
    - *Edge Cases / Potential Failures:*  
      - Missing required fields (form validation prevents submission)  
      - Invalid or empty prompt (handled by form required field)  
      - User selects unsupported image size (dropdown limits choices)  
    - *Version Requirements:* Uses Form Trigger v2.2 for enhanced form capabilities.

---

#### 2.2 AI Image Generation

- **Overview:**  
  This block sends the user inputs to the OpenAI API to generate an image using the GPT-Image-1 model.

- **Nodes Involved:**  
  - OpenAI Image Generation

- **Node Details:**

  - **OpenAI Image Generation**  
    - *Type and Role:* HTTP Request node; performs POST request to OpenAI’s image generation endpoint.  
    - *Configuration:*  
      - URL: `https://api.openai.com/v1/images/generations`  
      - Method: POST  
      - Authentication: Predefined OpenAI API credential (OAuth or API key)  
      - Body parameters:  
        - `model`: fixed to "gpt-image-1"  
        - `prompt`: dynamically set to user input `$json.Prompt`  
        - `n`: fixed to 1 (one image generated)  
        - `size`: dynamically set to user input `$json['Image size']`  
      - Sends headers and body as JSON.  
    - *Key Expressions/Variables:*  
      - Uses expressions to inject prompt and size from form input.  
    - *Input/Output Connections:*  
      - Input from "Prompt and options" node  
      - Output to "Convert to File" node  
    - *Edge Cases / Potential Failures:*  
      - API authentication errors (invalid or expired credentials)  
      - API rate limits or quota exceeded  
      - Network timeouts or connectivity issues  
      - Invalid prompt causing API rejection  
      - Unexpected API response structure  
    - *Version Requirements:* HTTP Request node v4.2 for advanced authentication and parameter handling.

---

#### 2.3 Output Delivery

- **Overview:**  
  This block converts the base64-encoded image data returned by the OpenAI API into a binary file and returns it to the user via the form interface for download.

- **Nodes Involved:**  
  - Convert to File  
  - Return to form

- **Node Details:**

  - **Convert to File**  
    - *Type and Role:* Convert To File node; transforms base64 image data into a binary file format.  
    - *Configuration:*  
      - Operation: toBinary  
      - Source property: `data[0].b64_json` (the base64 image string from API response)  
    - *Input/Output Connections:*  
      - Input from "OpenAI Image Generation" node  
      - Output to "Return to form" node  
    - *Edge Cases / Potential Failures:*  
      - Missing or malformed base64 data  
      - Conversion errors due to unexpected data format  
    - *Version Requirements:* Convert To File node v1.1 for base64 handling.

  - **Return to form**  
    - *Type and Role:* Form node; sends the generated image file back to the user as a form completion response.  
    - *Configuration:*  
      - Operation: completion  
      - Respond with: returnBinary (sends binary file)  
      - Completion title: "Result"  
      - Completion message: "Here is the created image:"  
      - Webhook ID assigned for response  
    - *Input/Output Connections:*  
      - Input from "Convert to File" node  
      - No output (end of workflow)  
    - *Edge Cases / Potential Failures:*  
      - Failure to send binary data to user  
      - Webhook response timeouts  
      - User disconnects before receiving response  
    - *Version Requirements:* Form node v1.

---

### 3. Summary Table

| Node Name             | Node Type            | Functional Role               | Input Node(s)          | Output Node(s)        | Sticky Note                                                                                                         |
|-----------------------|----------------------|------------------------------|------------------------|-----------------------|---------------------------------------------------------------------------------------------------------------------|
| Prompt and options     | Form Trigger         | Input Reception (User Input) | None                   | OpenAI Image Generation|                                                                                                                     |
| OpenAI Image Generation| HTTP Request         | AI Image Generation           | Prompt and options     | Convert to File       |                                                                                                                     |
| Convert to File        | Convert To File      | Convert base64 to binary file | OpenAI Image Generation| Return to form        |                                                                                                                     |
| Return to form         | Form                 | Output Delivery (Send image)  | Convert to File        | None                  |                                                                                                                     |
| Sticky Note           | Sticky Note          | Documentation / Workflow Info | None                   | None                  | # Welcome to my Simple OpenAI Image Generator Workflow! This workflow creates an image with the new OpenAI image model "GPT-Image-1" based on a form input. See https://docs.n8n.io/integrations/builtin/credentials/openai/ and contact https://www.linkedin.com/in/friedemann-schuetz |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger node**  
   - Name: `Prompt and options`  
   - Set Form Title: "OpenAI Image Generator"  
   - Add Form Fields:  
     - Text field labeled "Prompt", required, placeholder "Snow-covered mountain village in the Alps"  
     - Dropdown field labeled "Image size", required, options: "1024x1024", "1024x1536", "1536x1024"  
   - Save and note the webhook URL generated (used internally).

2. **Create an HTTP Request node**  
   - Name: `OpenAI Image Generation`  
   - Set HTTP Method: POST  
   - Set URL: `https://api.openai.com/v1/images/generations`  
   - Authentication: Select or create OpenAI API credentials (OAuth2 or API key)  
   - Body Parameters (JSON):  
     - `model`: "gpt-image-1" (static)  
     - `prompt`: Expression `{{$json.Prompt}}` (from form input)  
     - `n`: 1 (static)  
     - `size`: Expression `{{$json['Image size']}}` (from form input)  
   - Enable sending headers and body as JSON.

3. **Connect `Prompt and options` node output to `OpenAI Image Generation` node input.**

4. **Create a Convert To File node**  
   - Name: `Convert to File`  
   - Operation: toBinary  
   - Source Property: `data[0].b64_json` (path to base64 image data from OpenAI response)  

5. **Connect `OpenAI Image Generation` node output to `Convert to File` node input.**

6. **Create a Form node**  
   - Name: `Return to form`  
   - Operation: completion  
   - Respond With: returnBinary (to send the image file)  
   - Completion Title: "Result"  
   - Completion Message: "Here is the created image:"  
   - Note the webhook URL generated for response.

7. **Connect `Convert to File` node output to `Return to form` node input.**

8. **Activate the workflow.**

---

### 5. General Notes & Resources

| Note Content                                                                                                      | Context or Link                                                                                         |
|------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| OpenAI API access is required; see official n8n documentation for credential setup: https://docs.n8n.io/integrations/builtin/credentials/openai/ | Credential setup for OpenAI API                                                                        |
| Contact the workflow author for questions or support: https://www.linkedin.com/in/friedemann-schuetz             | Author contact via LinkedIn                                                                             |
| This workflow uses the new GPT-Image-1 model for image generation, which may require updated OpenAI API access.  | Model specification and API version considerations                                                    |
| The form trigger and form completion nodes require webhook URLs to be accessible externally for user interaction.| Network and security considerations for webhook exposure                                             |

---

This documentation fully describes the workflow structure, node configurations, and reproduction steps, enabling advanced users or AI agents to understand, modify, or recreate the workflow without referencing the original JSON.