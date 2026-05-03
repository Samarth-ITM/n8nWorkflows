Generate UGC Images from Form Submissions with Google Gemini and Telegram

https://n8nworkflows.xyz/workflows/generate-ugc-images-from-form-submissions-with-google-gemini-and-telegram-9922


# Generate UGC Images from Form Submissions with Google Gemini and Telegram

### 1. Workflow Overview

This workflow automates the creation of User-Generated Content (UGC) images by processing form submissions containing a character type selection (e.g., male or female) and an uploaded product image. It leverages Google Gemini AI (accessed through OpenRouter) to generate realistic, lifestyle-style advertising images based on the submitted inputs. The final AI-generated image is then sent as a photo message to a designated Telegram channel.

The workflow is logically divided into the following blocks:

- **1.1 Form Input Reception:** Receives user inputs via a web form with fields for character type and product image upload.
- **1.2 File Extraction and Data Merging:** Processes the uploaded image file, extracts it as Base64, and merges it with the character type data.
- **1.3 Data URL Creation:** Converts the Base64 image into a Data URL format for API compatibility.
- **1.4 AI Request Preparation and Execution:** Maps the data into the expected AI prompt format and sends a request to Google Gemini to generate a UGC image.
- **1.5 AI Response Processing:** Extracts the generated image data from the AI response and converts it back into a file.
- **1.6 Telegram Notification:** Sends the generated image as a photo message to a specified Telegram channel.

---

### 2. Block-by-Block Analysis

#### 2.1 Form Input Reception

- **Overview:**  
  Captures user submissions via a web form that includes a dropdown to select the character type and an upload field for the product image.

- **Nodes Involved:**  
  - Form submission with character type and image

- **Node Details:**  

  **Form submission with character type and image**  
  - Type: Form Trigger  
  - Role: Entry point that triggers the workflow upon form submission.  
  - Configuration:  
    - Form titled "Advertising image generator"  
    - Fields:  
      - File upload (single, required, accepts jpg/png/jpeg) labeled "Product image"  
      - Dropdown (required) labeled "Character model" with options "Homme" (Male) and "Femme" (Female)  
  - Inputs: None (trigger node)  
  - Outputs: Triggers two parallel outputs connected to "Merge the two data sets" and "Extract the form file" nodes.  
  - Edge Cases/Potential Failures:  
    - Missing required fields (form validation handles this)  
    - File upload issues (file format or size restrictions)  
  - No sub-workflows invoked.

#### 2.2 File Extraction and Data Merging

- **Overview:**  
  Extracts the uploaded image file as a Base64 string and merges it with the selected character type field into a single data object.

- **Nodes Involved:**  
  - Extract the form file  
  - Merge the two data sets

- **Node Details:**  

  **Extract the form file**  
  - Type: Extract From File  
  - Role: Converts the binary uploaded file into a Base64 string stored in JSON property "image_base64".  
  - Configuration:  
    - Operation: binaryToProperty  
    - Binary property name: "Image_du_produit" (French label for the uploaded image)  
    - Destination key: "image_base64"  
  - Inputs: Receives form submission binary data from the form trigger.  
  - Outputs: Passes JSON with Base64 image data.  
  - Edge Cases:  
    - Corrupt or unsupported file formats  
    - Large file sizes causing memory issues  

  **Merge the two data sets**  
  - Type: Merge  
  - Role: Combines two data streams — form field data and extracted Base64 image — by position, resulting in a single JSON object containing both character type and image data.  
  - Configuration:  
    - Mode: combine  
    - Combine by: position  
  - Inputs:  
    - Input 1: Form field data (character type)  
    - Input 2: Extracted Base64 image data  
  - Outputs: Single combined JSON object including both properties.  
  - Edge Cases:  
    - Input misalignment if inputs do not arrive simultaneously  
    - Missing fields in one input  

#### 2.3 Data URL Creation

- **Overview:**  
  Converts the Base64 image string into a Data URL format (`data:image/jpeg;base64,...`) and restructures the JSON to map the character type and image URL for downstream use.

- **Nodes Involved:**  
  - Creating a data URL  
  - Mapping

- **Node Details:**  

  **Creating a data URL**  
  - Type: Code (JavaScript)  
  - Role:  
    - Reads the character type ("Modèle de personnage") and Base64 image from the merged data  
    - Creates a Data URL string for the image that can be used by the AI API  
    - Outputs JSON with keys "modele_personnage" and "image_url"  
  - Key Expressions:  
    - Uses template literals to build Data URL: `data:image/jpeg;base64,${imageBase64}`  
  - Inputs: JSON containing Base64 image and character type  
  - Outputs: JSON with mapped fields for next step  
  - Edge Cases:  
    - Incorrect Base64 data causes broken Data URL  
    - Assumes JPEG format; if image is PNG, the mime type should be adjusted  

  **Mapping**  
  - Type: Set  
  - Role: Maps the "image_url" and "modele_personnage" fields into the exact property names required for the AI request payload.  
  - Configuration:  
    - Assigns "image" = "image_url"  
    - Assigns "personnage" = "modele_personnage"  
  - Inputs: Output from "Creating a data URL"  
  - Outputs: JSON with keys "image" and "personnage" for the AI request construction  
  - Edge Cases: Minimal risk; mainly data forwarding  

#### 2.4 AI Request Preparation and Execution

- **Overview:**  
  Constructs and sends a POST request to Google Gemini AI (via OpenRouter API) with the character type and image data to generate a UGC image.

- **Nodes Involved:**  
  - Google gemini

- **Node Details:**  

  **Google gemini**  
  - Type: HTTP Request  
  - Role: Sends the payload to the OpenRouter API endpoint for Google Gemini to generate a user-generated content image based on the inputs.  
  - Configuration:  
    - URL: `https://openrouter.ai/api/v1/chat/completions`  
    - Method: POST  
    - Authentication: Predefined credential of type OpenRouter API  
    - Body (JSON):  
      - Model: "google/gemini-2.5-flash-image-preview"  
      - Messages array: Contains a user role message with:  
        - Text prompting to create a realistic and engaging UGC image describing the character product with style and context details  
        - Embeds the image URL from the previous node  
    - Sends body as JSON  
  - Inputs: JSON with "image" and "personnage" fields mapped from previous node  
  - Outputs: AI response JSON containing generated images and metadata  
  - Edge Cases:  
    - API authentication failures (invalid or expired OpenRouter API key)  
    - API rate limits or timeouts  
    - Unexpected response structure changes  
    - Network connectivity issues  

#### 2.5 AI Response Processing

- **Overview:**  
  Extracts the generated image data from the AI response, removes the Data URL prefix, and converts the Base64 string back into a binary file format suitable for Telegram.

- **Nodes Involved:**  
  - Transform URL data  
  - Download the file

- **Node Details:**  

  **Transform URL data**  
  - Type: Code (JavaScript)  
  - Role:  
    - Accesses the first choice in the AI response JSON, extracts the Base64 image string from `choices[0].message.images[0].image_url.url`  
    - Removes the `data:image/png;base64,` prefix to isolate pure Base64 data  
    - Outputs JSON with a "data" field containing cleaned Base64 data  
  - Inputs: AI response JSON  
  - Outputs: JSON with Base64 string ready for file conversion  
  - Edge Cases:  
    - AI response missing expected keys or images array empty  
    - Unexpected data format or prefix variations  

  **Download the file**  
  - Type: Convert To File  
  - Role: Converts Base64 string from "data" property into a binary file object compatible with n8n and Telegram node.  
  - Configuration:  
    - Operation: toBinary  
    - Source property: "data"  
  - Inputs: JSON with Base64 data string  
  - Outputs: Binary file data ready to send  
  - Edge Cases:  
    - Corrupt Base64 data causing conversion failure  

#### 2.6 Telegram Notification

- **Overview:**  
  Sends the generated image as a photo message to a specified Telegram channel.

- **Nodes Involved:**  
  - Send a photo message

- **Node Details:**  

  **Send a photo message**  
  - Type: Telegram  
  - Role: Sends a photo message containing the AI-generated image to the Telegram channel "@assistantjaures".  
  - Configuration:  
    - Chat ID: "@assistantjaures"  
    - Operation: sendPhoto  
    - Sends binary data (the image file)  
    - Uses Telegram API credentials configured with OAuth2 or bot token  
  - Inputs: Binary image file from the previous node  
  - Outputs: Telegram API response confirming message sent  
  - Edge Cases:  
    - Invalid chat ID or bot permissions  
    - Telegram API errors or rate limits  
    - Network issues  

---

### 3. Summary Table

| Node Name                          | Node Type              | Functional Role                              | Input Node(s)                           | Output Node(s)                | Sticky Note                                                                                                                    |
|-----------------------------------|------------------------|----------------------------------------------|---------------------------------------|------------------------------|-------------------------------------------------------------------------------------------------------------------------------|
| Form submission with character type and image | Form Trigger           | Entry point; captures form data (image + character) | None                                  | Merge the two data sets, Extract the form file | ## Form Submission with Character Type and Image                                                                              |
| Extract the form file              | Extract From File      | Converts uploaded image binary to Base64     | Form submission with character type and image | Merge the two data sets        | ## Extract the Form File & Merge the Two Data Sets                                                                             |
| Merge the two data sets            | Merge                  | Combines character type and image data       | Form submission with character type and image, Extract the form file | Creating a data URL           | ## Extract the Form File & Merge the Two Data Sets                                                                             |
| Creating a data URL               | Code                   | Creates Data URL from Base64 image, maps fields | Merge the two data sets               | Mapping                      | ## NanoBanana UGC image generator                                                                                              |
| Mapping                          | Set                    | Maps fields for AI API payload                 | Creating a data URL                   | Google gemini                | ## NanoBanana UGC image generator                                                                                              |
| Google gemini                   | HTTP Request           | Sends AI image generation request to Google Gemini | Mapping                             | Transform URL data            | ## NanoBanana UGC image generator                                                                                              |
| Transform URL data               | Code                   | Extracts and cleans Base64 image from AI response | Google gemini                      | Download the file            | ## UGCImage generated sent to Telegram                                                                                        |
| Download the file               | Convert To File        | Converts Base64 to binary file for Telegram   | Transform URL data                   | Send a photo message         | ## UGCImage generated sent to Telegram                                                                                        |
| Send a photo message            | Telegram               | Sends the generated image to Telegram channel | Download the file                   | None                         | ## UGCImage generated sent to Telegram                                                                                        |
| Sticky Note1                    | Sticky Note            | Label for form submission block                | None                                 | None                         | ## Form Submission with Character Type and Image                                                                              |
| Sticky Note2                    | Sticky Note            | Label for extraction and merging               | None                                 | None                         | ## Extract the Form File & Merge the Two Data Sets                                                                             |
| Sticky Note3                    | Sticky Note            | Label for AI image generation                   | None                                 | None                         | ## NanoBanana UGC image generator                                                                                              |
| Sticky Note4                    | Sticky Note            | Label for Telegram notification                 | None                                 | None                         | ## UGCImage generated sent to Telegram                                                                                        |
| Sticky Note                    | Sticky Note            | Workflow overview and instructions              | None                                 | None                         | ## This workflow automates the creation of UGC (User-Generated Content) images by allowing users to submit a form containing... |
| Sticky Note6                   | Sticky Note            | Detailed workflow description, usage, and help | None                                 | None                         | ## This workflow automates the process of generating personalized UGC images based on form submissions...                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger node**  
   - Type: Form Trigger  
   - Configure form: Title "Advertising image generator"  
   - Add two fields:  
     - File upload field labeled "Product image" (required, single file, accept .jpg, .jpeg, .png)  
     - Dropdown labeled "Character model" (required, options: "Homme", "Femme")  
   - Save and connect as the starting point of the workflow.

2. **Create Extract From File node**  
   - Type: Extract From File  
   - Configure: Operation "binaryToProperty"  
   - Binary property name: "Image_du_produit" (matches form file field label)  
   - Destination key: "image_base64"  
   - Connect input from the Form Trigger node (second output connection).

3. **Create Merge node**  
   - Type: Merge  
   - Configure: Mode "combine"  
   - Combine by "position"  
   - Connect two inputs:  
     - First input from Form Trigger (first output connection) (character type data)  
     - Second input from Extract From File (Base64 image data)  
   - Output will combine character and image data.

4. **Create Code node ("Creating a data URL")**  
   - Type: Code  
   - JavaScript to:  
     - Extract "Modèle de personnage" and "image_base64" from merged data  
     - Create Data URL string: `data:image/jpeg;base64,${imageBase64}`  
     - Return JSON with keys:  
       - "modele_personnage" = character model  
       - "image_url" = Data URL string  
   - Connect input from Merge node.

5. **Create Set node ("Mapping")**  
   - Type: Set  
   - Assign two fields:  
     - "image" = value of "image_url" from previous node  
     - "personnage" = value of "modele_personnage" from previous node  
   - Connect input from Code node.

6. **Create HTTP Request node ("Google gemini")**  
   - Type: HTTP Request  
   - Configure:  
     - URL: `https://openrouter.ai/api/v1/chat/completions`  
     - Method: POST  
     - Authentication: Use saved OpenRouter API credential  
     - Body type: JSON  
     - Body content: construct JSON with:  
       - model: `"google/gemini-2.5-flash-image-preview"`  
       - messages: array containing one user role message with content:  
         - Text prompt describing UGC image creation incorporating `{{$json.personnage}}`  
         - Image URL embedded using `{{$json.image}}`  
   - Connect input from Set node.

7. **Create Code node ("Transform URL data")**  
   - Type: Code  
   - JavaScript to:  
     - Extract Base64 image from AI response path: `choices[0].message.images[0].image_url.url`  
     - Remove `data:image/png;base64,` prefix  
     - Return JSON with "data" field containing clean Base64 string  
   - Connect input from HTTP Request node.

8. **Create Convert To File node ("Download the file")**  
   - Type: Convert To File  
   - Configure:  
     - Operation: toBinary  
     - Source property: "data"  
   - Connect input from previous Code node.

9. **Create Telegram node ("Send a photo message")**  
   - Type: Telegram  
   - Operation: sendPhoto  
   - Chat ID: "@assistantjaures" (or your own Telegram channel)  
   - Enable binary data sending  
   - Use configured Telegram API credentials (bot token or OAuth2)  
   - Connect input from Convert To File node.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                               | Context or Link                                                                                                     |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------|
| This workflow automates the creation of UGC images by processing form submissions containing character type and image, sending to Google Gemini via OpenRouter, and posting results to Telegram.                                                                                                                                                                                           | Workflow Overview (Sticky Note)                                                                                    |
| Designed for marketers, AI creators, content teams, or community platforms to enable no-code AI-enhanced content generation and instant Telegram publishing. Requires OpenRouter API key and Telegram Bot connected to a channel.                                                                                                                                                             | Detailed Workflow Description (Sticky Note)                                                                       |
| Contact for consulting and support: LinkedIn - https://www.linkedin.com/in/jaures-nya-83a033270/ , YouTube - https://www.youtube.com/@jauresnya , Skool - https://www.skool.com/gaia-4903/about?ref=e0430e4c35b645ac8976b952768e9d55                                                                                                                                                             | Support and Contact Links                                                                                          |
| The AI prompt and image generation style are curated to produce lifestyle, natural, authentic UGC photos that feel spontaneous and realistic, using soft colors and simple backgrounds. Adjust prompt text in the HTTP Request node if needed to customize image style.                                                                                                                      | AI Prompt Configuration                                                                                             |
| OpenRouter API offers an abstraction layer for Google Gemini API access; ensure your API key is valid and has sufficient quota.                                                                                                                                                                                                                                                             | API Usage Note                                                                                                     |
| Telegram bot must have permission to post in the target chat/channel; chat ID must be accurate and consistent with bot’s access rights.                                                                                                                                                                                                                                                    | Telegram Integration Note                                                                                           |

---

**Disclaimer:**  
The provided text is derived exclusively from an automated workflow created with n8n, a no-code integration and automation platform. It strictly complies with current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.