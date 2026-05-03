Generate Product Mockups with Gemini 2.5 Flash Image via OpenRouter

https://n8nworkflows.xyz/workflows/generate-product-mockups-with-gemini-2-5-flash-image-via-openrouter-8194


# Generate Product Mockups with Gemini 2.5 Flash Image via OpenRouter

---
  
### 1. Workflow Overview

This n8n workflow, titled **"ProBanana: AI Product Mockup Generator using Gemini 2.5 Flash Image (Nano Banana)"**, automates the generation of AI-enhanced product mockups. It is designed to take user-uploaded images—a product image and a template/playground image—along with a textual prompt describing the desired modification or effect. The workflow processes these inputs, encodes the images in Base64, and sends a multimodal request to the OpenRouter API utilizing Google’s Gemini 2.5 Flash Image model. The AI model then generates a new image based on the prompt and the two input images, returning a final product mockup.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Capturing user input via a form submission (product image, template image, and prompt).
- **1.2 Data Preparation:** Extracting Base64 data from the uploaded images and assembling all input data into a structured JSON payload.
- **1.3 AI Processing:** Sending the assembled data as a multimodal prompt to the OpenRouter API using the Gemini 2.5 model.
- **1.4 Output Processing:** Extracting the returned image from the API response, converting it to a binary file suitable for downstream use or download.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block receives user inputs via a web form including two images and a textual prompt. It initiates the workflow when the form is submitted.

- **Nodes Involved:**  
  - On form submission

- **Node Details:**  

  | Node Name         | Details                                                                                                 |
  |-------------------|---------------------------------------------------------------------------------------------------------|
  | On form submission | - Type: Form Trigger node, starts workflow on form submit<br>- Configured with a form titled "Nano Banana"<br>- Fields: <ul><li>Upload your product image (required, jpg/png/jpeg)</li><li>Upload your template/playground image (required, jpg/png/jpeg)</li><li>Prompt text (required)</li></ul><br>- Outputs form data including binary files for images and text fields<br>- Input connections: none (trigger node)<br>- Output connections: User Asset Base64<br>- Failure considerations: Form validation errors, unsupported file types, missing fields |

---

#### 2.2 Data Preparation

- **Overview:**  
  Extracts Base64-encoded data from the uploaded image files and consolidates the prompt and image data into a single JSON object for the API request.

- **Nodes Involved:**  
  - User Asset Base64  
  - Template Base64  
  - Assemble Final Data

- **Node Details:**  

  | Node Name          | Details                                                                                                  |
  |--------------------|----------------------------------------------------------------------------------------------------------|
  | User Asset Base64   | - Type: Extract From File node<br>- Converts binary product image file to Base64 string stored in JSON property `data`<br>- Input: binary file from "On form submission" node's "Upload your product image"<br>- Output: JSON with Base64 string<br>- Output connected to "Template Base64"<br>- Failure cases: corrupted files, unsupported formats |
  | Template Base64     | - Type: Extract From File node<br>- Converts binary template image file to Base64 string stored in JSON property `data`<br>- Input: binary file from "User Asset Base64" node (chained input)<br>- Output: JSON with Base64 string<br>- Output connected to "Assemble Final Data"<br>- Failure cases: corrupted files, unsupported formats |
  | Assemble Final Data | - Type: Set node<br>- Combines the prompt text and the two Base64 image strings into a single JSON object with keys: `promptText`, `assetBase64`, `templateBase64`<br>- Uses expressions to extract values:<br>  - `promptText` from form field "Pls describe what you want to do?"<br>  - `assetBase64` from "User Asset Base64" node JSON data<br>  - `templateBase64` from "Template Base64" node JSON data<br>- Output connected to "HTTP Request"<br>- Failure considerations: expression errors if prior nodes lack expected data |

---

#### 2.3 AI Processing

- **Overview:**  
  Sends a POST request to the OpenRouter API with a multimodal chat completion containing the prompt text and both images encoded as Base64 data URLs. Uses the Gemini 2.5 Flash Image model to generate an enhanced product mockup image.

- **Nodes Involved:**  
  - HTTP Request

- **Node Details:**  

  | Node Name     | Details                                                                                                   |
  |---------------|-----------------------------------------------------------------------------------------------------------|
  | HTTP Request  | - Type: HTTP Request node<br>- Method: POST<br>- URL: `https://openrouter.ai/api/v1/chat/completions`<br>- Auth: Uses predefined OpenRouter credentials with OAuth2 (credential `OpenRouter account`)<br>- Body: JSON including:<br>  - `model`: `"google/gemini-2.5-flash-image-preview:free"`<br>  - `messages`: array of multimodal message parts:<br>    - Text part with user prompt<br>    - Image URL parts with Base64 data URLs of product and template images<br>- Sends JSON body with expressions injecting dynamic data<br>- Output connected to "Edit Fields"<br>- Failure considerations: API auth errors, rate limiting, invalid Base64, malformed JSON, network timeouts |

---

#### 2.4 Output Processing

- **Overview:**  
  Extracts the generated image's Base64 string from the API response, converts it to binary format, and prepares it for final use or download.

- **Nodes Involved:**  
  - Edit Fields  
  - Convert to File

- **Node Details:**  

  | Node Name     | Details                                                                                                 |
  |---------------|---------------------------------------------------------------------------------------------------------|
  | Edit Fields   | - Type: Set node<br>- Extracts the Base64 string of the generated image from: `choices[0].message.images[0].image_url.url`<br>- Splits the URL string by comma to isolate the Base64 part<br>- Stores the pure Base64 string in JSON property `outputfile`<br>- Input: API response JSON from "HTTP Request"<br>- Output: JSON with Base64 string<br>- Output connected to "Convert to File"<br>- Failure cases: missing or malformed image URL, unexpected API response structure |
  | Convert to File | - Type: Convert To File node<br>- Converts the Base64 string in `outputfile` JSON property to binary data<br>- Output: binary image file suitable for download or further processing<br>- Input: JSON from "Edit Fields"<br>- Failure cases: invalid Base64 data |

---

### 3. Summary Table

| Node Name           | Node Type           | Functional Role                   | Input Node(s)           | Output Node(s)        | Sticky Note                                                                                                    |
|---------------------|---------------------|---------------------------------|------------------------|-----------------------|---------------------------------------------------------------------------------------------------------------|
| On form submission   | Form Trigger        | Receives user input via form     | —                      | User Asset Base64      | ## Workflow Initiation<br>### Form to take the user asset image and the template image along with the prompt(describing of what is to be done) |
| User Asset Base64    | Extract From File   | Converts product image to Base64 | On form submission     | Template Base64        | ## Product (User Asset)<br>![image](https://n3wstorage.b-cdn.net/n3witalia/tshirt.jpg)                         |
| Template Base64      | Extract From File   | Converts template image to Base64| User Asset Base64      | Assemble Final Data    | ## Template Model<br>![image](https://n3wstorage.b-cdn.net/n3witalia/model.jpg)                               |
| Assemble Final Data  | Set                 | Combines prompt and images data  | Template Base64        | HTTP Request           |                                                                                                               |
| HTTP Request        | HTTP Request        | Sends data to OpenRouter API     | Assemble Final Data    | Edit Fields            | ## Core AI Generation Step<br>### Sends a `POST` request to the OpenRouter API with a multimodal prompt containing:<br>### 1. The user's text instruction.<br>### 2. The user's asset image (Base64).<br>### 3. The template image (Base64).<br><br>### Uses the `google/gemini-2.5-flash-image-preview` model to generate the new image. |
| Edit Fields         | Set                 | Extracts generated image Base64  | HTTP Request           | Convert to File        |                                                                                                               |
| Convert to File     | Convert To File     | Converts Base64 string to binary | Edit Fields            | —                      |                                                                                                               |
| Sticky Note2        | Sticky Note         | Describes workflow initiation    | —                      | —                      | ## Workflow Initiation<br>### Form to take the user asset image and the template image along with the prompt(describing of what is to be done) |
| Sticky Note3        | Sticky Note         | Shows example final output image | —                      | —                      | ## Final Output<br>![image](https://n3wstorage.b-cdn.net/n3witalia/result_sport.jpeg)                         |
| Sticky Note4        | Sticky Note         | Describes AI generation step     | —                      | —                      | ## Core AI Generation Step<br>### Sends a `POST` request to the OpenRouter API with a multimodal prompt containing:<br>### 1. The user's text instruction.<br>### 2. The user's asset image (Base64).<br>### 3. The template image (Base64).<br><br>### Uses the `google/gemini-2.5-flash-image-preview` model to generate the new image. |
| Sticky Note5        | Sticky Note         | Shows template model example     | —                      | —                      | ## Template Model<br>![image](https://n3wstorage.b-cdn.net/n3witalia/model.jpg)                               |
| Sticky Note6        | Sticky Note         | Shows product example image      | —                      | —                      | ## Product (User Asset)<br>![image](https://n3wstorage.b-cdn.net/n3witalia/tshirt.jpg)                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger Node**  
   - Type: Form Trigger  
   - Name: `On form submission`  
   - Configure form title: `"Nano Banana"`  
   - Add form fields:  
     - File field labeled "Upload your product image" (required, accept `.jpg, .png, .jpeg`)  
     - File field labeled "Upload your template/playground image" (required, accept `.jpg, .png, .jpeg`)  
     - Text field labeled "Pls describe what you want to do?" (required, placeholder: "Prompt")  
   - This node will start the workflow when the form is submitted.

2. **Add Extract From File Node for Product Image**  
   - Type: Extract From File  
   - Name: `User Asset Base64`  
   - Operation: `binaryToProperty`  
   - Binary Property Name: `Upload_your_product_image` (exactly matching the form field name)  
   - Connect input from `On form submission`.

3. **Add Extract From File Node for Template Image**  
   - Type: Extract From File  
   - Name: `Template Base64`  
   - Operation: `binaryToProperty`  
   - Binary Property Name: `Upload_your_template_playground_image`  
   - Connect input from `User Asset Base64`.

4. **Add Set Node to Assemble Final Data**  
   - Type: Set  
   - Name: `Assemble Final Data`  
   - Add fields:  
     - `promptText`: Set as expression to `{{$node["On form submission"].json["Pls describe what you want to do?"]}}`  
     - `assetBase64`: Set as expression to `{{$node["User Asset Base64"].json["data"]}}`  
     - `templateBase64`: Set as expression to `{{$json["data"]}}` (from current node input)  
   - Connect input from `Template Base64`.

5. **Add HTTP Request Node to Call OpenRouter API**  
   - Type: HTTP Request  
   - Name: `HTTP Request`  
   - Method: POST  
   - URL: `https://openrouter.ai/api/v1/chat/completions`  
   - Authentication: Use predefined OpenRouter API credentials with OAuth2 (configure credentials beforehand)  
   - Request Body (JSON):  
     ```json
     {
       "model": "google/gemini-2.5-flash-image-preview:free",
       "messages": [
         {
           "role": "user",
           "content": [
             { "type": "text", "text": "{{ $json.promptText }}" },
             { "type": "image_url", "image_url": { "url": "data:image/jpeg;base64,{{ $json.assetBase64 }}" } },
             { "type": "image_url", "image_url": { "url": "data:image/jpeg;base64,{{ $json.templateBase64 }}" } }
           ]
         }
       ]
     }
     ```  
   - Set "Send Body" to true and "Specify Body" as JSON.  
   - Connect input from `Assemble Final Data`.

6. **Add Set Node to Extract Final Image Base64**  
   - Type: Set  
   - Name: `Edit Fields`  
   - Add field `outputfile`: expression:  
     ```js
     {{ $json.choices[0].message.images[0].image_url.url.split(",")[1] }}
     ```  
   - Connect input from `HTTP Request`.

7. **Add Convert To File Node to Convert Base64 to Binary**  
   - Type: Convert To File  
   - Name: `Convert to File`  
   - Operation: `toBinary`  
   - Source Property: `outputfile`  
   - Connect input from `Edit Fields`.

8. **Add Sticky Notes (Optional for Documentation/Visual Aid)**  
   - Add sticky notes with explanatory content covering:  
     - Workflow initiation (form inputs)  
     - Product asset example image  
     - Template model example image  
     - Core AI generation explanation  
     - Final output example image

---

### 5. General Notes & Resources

| Note Content                                                                                                            | Context or Link                                                    |
|-------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------|
| Workflow uses the OpenRouter API with the Gemini 2.5 Flash Image model for multimodal AI generation combining text and images. | OpenRouter API documentation: https://openrouter.ai              |
| Sample images and model references are hosted at https://n3wstorage.b-cdn.net/n3witalia/                               | Images linked in sticky notes for demonstration purposes          |
| The design assumes valid image uploads conforming to accepted formats (.jpg, .png, .jpeg) and proper Base64 encoding.   | Be mindful of file size limits and API rate limits during usage  |
| Authentication requires setting up OpenRouter OAuth2 credentials in n8n credentials prior to running the workflow.       | See n8n credential setup documentation for OAuth2 APIs            |

---

**Disclaimer:** The provided text and workflow are generated exclusively by an automated n8n workflow. All data processed is legal and public; no illegal, offensive, or protected content is included. The workflow respects all current content policies.