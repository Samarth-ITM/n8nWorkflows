Generate Fashion Model Product Ads with Gemini AI via OpenRouter

https://n8nworkflows.xyz/workflows/generate-fashion-model-product-ads-with-gemini-ai-via-openrouter-8193


# Generate Fashion Model Product Ads with Gemini AI via OpenRouter

### 1. Workflow Overview

This workflow automates the generation of high-quality, photorealistic fashion advertisement images using Google’s Gemini AI model accessed via OpenRouter. It takes a user-submitted product image and a selected character model (Male/Female) from a form, generates an image of a fashion model actively using the product in a natural setting, and returns the generated image for download.

The workflow consists of these logical blocks:

- **1.1 Input Reception:** Captures product images and model preferences via a web form.
- **1.2 Image Preprocessing:** Extracts and converts the uploaded product image into a format suitable for AI input.
- **1.3 AI Image Generation:** Sends a structured prompt and the product image to Gemini AI through OpenRouter, requesting a photorealistic advertisement image.
- **1.4 Postprocessing and Output:** Processes the AI response, converts the generated image data to a downloadable file, and delivers it via a completion form.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
This block captures user input via a web form where users upload a product image and select a character model type (Male or Female). It triggers the workflow on form submission.

**Nodes Involved:**  
- On form submission

**Node Details:**  

- **On form submission**  
  - *Type:* Form Trigger  
  - *Role:* Entry point that listens for form submissions containing product image and character model choice.  
  - *Configuration:*  
    - Form titled "Advertise Image Generator" with a file upload field labeled "Product Image" (accepting .jpg, .png, required) and a dropdown labeled "Character Model" with options Male or Female.  
    - Button labeled "Generate ✨".  
    - Form description explains it converts simple product images into high-quality advertisement images.  
  - *Inputs:* HTTP webhook call on form submission  
  - *Outputs:* Raw form data including binary file data for the image and JSON for the dropdown selection.  
  - *Edge cases:*  
    - Missing or unsupported file type uploads  
    - Empty selection for character model (though not marked required)  
    - Webhook connectivity or timeout failures  

---

#### 1.2 Image Preprocessing

**Overview:**  
Processes the uploaded product image from binary to base64, then formats it to prepare for inclusion in the AI prompt.

**Nodes Involved:**  
- Extract from File  
- Code

**Node Details:**  

- **Extract from File**  
  - *Type:* Extract from File  
  - *Role:* Converts the uploaded binary image file (from the form) to a base64-encoded string stored in JSON.  
  - *Configuration:* Operation "binaryToProperty" on the binary property "Product_Image" (matches form field name).  
  - *Input:* Binary file from form trigger node  
  - *Output:* JSON containing base64 data string  
  - *Edge cases:*  
    - Corrupted or unreadable image file  
    - Binary property missing or misnamed  

- **Code**  
  - *Type:* Code (JavaScript)  
  - *Role:* Wraps the base64 image data into a properly formatted Data URL string (`data:image/png;base64,...`) to be used by the AI model.  
  - *Configuration:* Custom JS snippet that maps input items, extracts base64 data, and constructs the Data URL string.  
  - *Key expressions:*  
    ```js
    const base64Url = item?.json?.data;
    const url = `data:image/png;base64,${base64Url}`;
    return { url };
    ```  
  - *Input:* JSON with base64 image data  
  - *Output:* JSON with a single field `url` containing the data URL string  
  - *Edge cases:*  
    - Missing or malformed base64 string causing invalid Data URL  
    - JS runtime errors  

---

#### 1.3 AI Image Generation

**Overview:**  
Calls OpenRouter API with a chat completion request to Google Gemini model, providing a prompt and the product image URL to generate a photorealistic fashion model advertisement image.

**Nodes Involved:**  
- Nano 🍌 (HTTP Request)  
- Edit Fields

**Node Details:**  

- **Nano 🍌**  
  - *Type:* HTTP Request  
  - *Role:* Sends POST request to OpenRouter’s chat completions endpoint to invoke Gemini AI model for image generation.  
  - *Configuration:*  
    - URL: `https://openrouter.ai/api/v1/chat/completions`  
    - Method: POST  
    - JSON payload structured as a chat completion with:  
      - Model: `"google/gemini-2.5-flash-image-preview:free"`  
      - Messages: user role message containing a prompt that instructs the model to generate a photorealistic image of the selected character model actively using the product. The prompt dynamically injects the "Character Model" choice from the form.  
      - Includes the product image as an embedded `image_url` using the Data URL generated previously.  
    - Authorization: Bearer token via OpenRouter API key credential  
  - *Input:* JSON with Data URL of the product image and character model choice  
  - *Output:* AI response JSON containing generated image URLs in nested properties: `choices[0].message.images[0].image_url.url`  
  - *Key expressions:* Template expressions inject dynamic values from prior nodes.  
  - *Credential requirements:* Predefined OpenRouter API credentials configured with API key.  
  - *Edge cases:*  
    - API authentication errors (invalid/missing API key)  
    - API rate limits or timeouts  
    - Unexpected response structure or missing image URLs  
    - Model unavailability or errors from OpenRouter  

- **Edit Fields**  
  - *Type:* Set  
  - *Role:* Extracts and separates key components from the returned image URL string for subsequent file conversion.  
  - *Configuration:*  
    - Extract full Data URL string into `data` variable  
    - Extract base64 portion (after comma) into `base`  
    - Extract MIME type (from Data URL prefix) into `mime`  
    - Sets a static `fileName` with value `.png` (likely used downstream)  
  - *Input:* AI response JSON from Nano 🍌 node  
  - *Output:* JSON with separated fields for base64 data and MIME type  
  - *Edge cases:*  
    - Unexpected URL formatting causing failed string splits  
    - Missing or null URL field  

---

#### 1.4 Postprocessing and Output

**Overview:**  
Converts the extracted base64 image data into a binary file and presents the generated image to the user via a completion form for automatic download.

**Nodes Involved:**  
- Convert to File  
- Form

**Node Details:**  

- **Convert to File**  
  - *Type:* Convert to File  
  - *Role:* Converts base64 string (stored in `base`) into a binary file format to be downloadable.  
  - *Configuration:*  
    - Operation: toBinary  
    - Source property: `base` (from Edit Fields node)  
  - *Input:* JSON with base64 image data string  
  - *Output:* Binary file data representing the generated image  
  - *Edge cases:*  
    - Invalid base64 string causing conversion failure  
    - Binary property naming conflicts  

- **Form**  
  - *Type:* Form (Completion)  
  - *Role:* Sends back a completion response to the user indicating that image generation is completed, delivering the generated image in a downloadable format.  
  - *Configuration:*  
    - Respond with binary data for automatic download  
    - Title: "Image Generation Completed"  
    - Message: "Generated Image Will be Automatically Downloaded in Your Device"  
  - *Input:* Binary image file from Convert to File node  
  - *Output:* HTTP response to client completing the form interaction  
  - *Edge cases:*  
    - Client-side download failures (browser issues)  
    - Response size limits or timeouts  

---

### 3. Summary Table

| Node Name           | Node Type           | Functional Role                           | Input Node(s)           | Output Node(s)           | Sticky Note                                                                                                                           |
|---------------------|---------------------|-----------------------------------------|------------------------|--------------------------|-------------------------------------------------------------------------------------------------------------------------------------|
| On form submission  | Form Trigger        | Receives product image and model choice | -                      | Extract from File         | --- Setup Guide: Steps to configure form, OpenRouter API, image processing, and output form. See sticky note for full details.     |
| Extract from File   | Extract from File   | Converts uploaded binary image to base64| On form submission     | Code                     |                                                                                                                                     |
| Code                | Code                | Formats base64 data to Data URL string  | Extract from File       | Nano 🍌                   |                                                                                                                                     |
| Nano 🍌             | HTTP Request        | Calls Gemini AI via OpenRouter to generate image | Code                  | Edit Fields              |                                                                                                                                     |
| Edit Fields         | Set                 | Extracts base64 and MIME type from AI response | Nano 🍌              | Convert to File           |                                                                                                                                     |
| Convert to File     | Convert to File     | Converts base64 string to binary file   | Edit Fields             | Form                     |                                                                                                                                     |
| Form                | Form (Completion)   | Delivers generated image to user for download | Convert to File      | -                        |                                                                                                                                     |
| Sticky Note         | Sticky Note         | Workflow title display                   | -                      | -                        | # Nano 🍌                                                                                                                           |
| Sticky Note1        | Sticky Note         | Setup guide with detailed instructions  | -                      | -                        | Setup Guide with instructions and link to OpenRouter: https://openrouter.ai/                                                        |
| Sticky Note3        | Sticky Note         | YouTube tutorial link                    | -                      | -                        | Start here: Step by Step Youtube Tutorial with embedded thumbnail and link: https://youtu.be/UGf01FYaAzY?si=xaO5N1xeXotZ0tYN      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger node**  
   - Name: `On form submission`  
   - Configure webhook with these form fields:  
     - File upload field labeled "Product Image" (accepts `.jpg, .png`, required, single file)  
     - Dropdown labeled "Character Model" with options: Male, Female  
   - Set button label to "Generate ✨"  
   - Form title: "Advertise Image Generator"  
   - Form description: "Convert Simple Product Images to High Quality Advertise Images"

2. **Add Extract from File node**  
   - Set operation: `binaryToProperty`  
   - Binary property name: `Product_Image` (matching the form upload field)  
   - Connect input from `On form submission` node

3. **Add a Code node**  
   - Insert this JavaScript code to convert base64 to Data URL:  
   ```js
   const items = $input.all();
   const updatedItems = items.map((item) => {
     const base64Url = item?.json?.data;
     const url = `data:image/png;base64,${base64Url}`;
     return { url };
   });
   return updatedItems;
   ```  
   - Connect input from `Extract from File`

4. **Add HTTP Request node** (name it `Nano 🍌`)  
   - Method: POST  
   - URL: `https://openrouter.ai/api/v1/chat/completions`  
   - Authentication: Use OpenRouter API credentials, inserting your API key  
   - Headers: Authorization: Bearer `$OPENROUTER_API_KEY`  
   - Body content type: JSON  
   - Body (raw JSON with expressions):  
   ```json
   {
     "model": "google/gemini-2.5-flash-image-preview:free",
     "messages": [
       {
         "role": "user",
         "content": [
           {
             "type": "text",
             "text": "This image is a picture of a product. Generate a new, photorealistic image of a {{$('On form submission').item.json['Character Model']}} fashion model actively using this exact product in a natural setting. The {{$('On form submission').item.json['Character Model']}} model should look happy and engaged with the product. Pay close attention to the product's design, colors, and details from the reference image."
           },
           {
             "type": "image_url",
             "image_url": {
               "url": "{{ $json.url }}"
             }
           }
         ]
       }
     ]
   }
   ```  
   - Connect input from `Code` node

5. **Add Set node** (name it `Edit Fields`)  
   - Assign variables:  
     - `data` = full image URL string from AI response: `={{ $json.choices[0].message.images[0].image_url.url }}`  
     - `base` = base64 part after comma: `={{ $json.choices[0].message.images[0].image_url.url.split(',')[1] }}`  
     - `mime` = MIME type extracted from Data URL prefix: `={{ $json.choices[0].message.images[0].image_url.url.split(';')[0].split(':')[1] }}`  
     - `fileName` = `.png` (static value)  
   - Connect input from `Nano 🍌`

6. **Add Convert to File node**  
   - Operation: `toBinary`  
   - Source property: `base`  
   - Connect input from `Edit Fields`

7. **Add Form node**  
   - Operation: Completion  
   - Respond with: Binary (automatic download)  
   - Completion title: "Image Generation Completed"  
   - Completion message: "Generated Image Will be Automatically Downloaded in Your Device"  
   - Connect input from `Convert to File`

8. **(Optional) Add Sticky Notes for documentation and branding**  
   - Title note: "# Nano 🍌"  
   - Setup guide note with instructions and OpenRouter link: https://openrouter.ai/  
   - YouTube tutorial note with embedded thumbnail and link: https://youtu.be/UGf01FYaAzY?si=xaO5N1xeXotZ0tYN  

9. **Credentials Setup**  
   - Configure OpenRouter API key credential (named e.g., "OpenRouter account") and assign it in the HTTP Request node.  
   - Ensure correct API key permissions and quota.

10. **Test**  
    - Publish the workflow and submit the form with a product image and character selection.  
    - Confirm that the generated image downloads automatically after processing.

---

### 5. General Notes & Resources

| Note Content                                                                                                     | Context or Link                                                                                                  |
|-----------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| Setup Guide with detailed instructions for form setup, OpenRouter API key configuration, and testing.          | https://openrouter.ai/                                                                                           |
| YouTube tutorial video demonstrating step-by-step workflow usage and setup.                                     | https://youtu.be/UGf01FYaAzY?si=xaO5N1xeXotZ0tYN                                                                |
| Workflow branding uses “Nano 🍌” as a project codename/title.                                                    | Displayed in sticky note at the top of the workflow.                                                            |
| Uses Google Gemini 2.5 flash image preview model via OpenRouter for AI image generation.                        | OpenRouter API documentation: https://openrouter.ai/docs                                                         |

---

**Disclaimer:**  
The provided text is derived exclusively from an automated workflow built with n8n, a workflow automation tool. It strictly complies with content policies and contains no illegal, offensive, or protected material. All data processed is legal and public.