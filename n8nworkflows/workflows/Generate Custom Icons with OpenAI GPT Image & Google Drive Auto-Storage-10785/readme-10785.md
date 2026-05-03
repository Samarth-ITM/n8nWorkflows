Generate Custom Icons with OpenAI GPT Image & Google Drive Auto-Storage

https://n8nworkflows.xyz/workflows/generate-custom-icons-with-openai-gpt-image---google-drive-auto-storage-10785


# Generate Custom Icons with OpenAI GPT Image & Google Drive Auto-Storage

---

### 1. Workflow Overview

This workflow automates the process of generating custom icons based on user input through a web form. It is designed for UI/UX designers, developers, or teams who need quick, high-quality icons tailored to specific subjects and styles without manual graphic design effort.

The workflow is structured into the following logical blocks:

- **1.1 Input Reception:** Captures user input from a form, including icon subject, style, and background.
- **1.2 Prompt Generation:** Uses an advanced GPT-5 language model to create an optimized, structured JSON prompt for image generation based on the user input.
- **1.3 AI Image Rendering:** Generates a high-quality 400×400 PNG icon with transparency using OpenAI's GPT Image model.
- **1.4 Cloud Storage Upload:** Automatically uploads the generated icon image to a predefined Google Drive folder, providing accessible links.
- **1.5 User Feedback Display:** Presents a confirmation form with preview, download link, and an option to generate another icon.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block triggers the workflow upon user submission of the "Icon Generator" form, collecting three crucial inputs: the icon subject, icon style, and background. It ensures structured and reliable input data for subsequent processing.

- **Nodes Involved:**  
  - When form is submitted

- **Node Details:**

  - **Node Name:** When form is submitted  
    - **Type:** Form Trigger  
    - **Role:** Entry point of the workflow; listens for user form submissions.  
    - **Configuration:**  
      - Form Title: "Icon Generator"  
      - Fields:  
        - Icon Subject (text dropdown with example placeholders)  
        - Icon Style (dropdown with 18 detailed style options like Flat, Outline, Neumorphic, etc.)  
        - Icon Background (text input)  
    - **Inputs:** None (trigger)  
    - **Outputs:** JSON containing user inputs  
    - **Edge Cases & Failure Types:**  
      - Form not submitted or incomplete data  
      - User cancels or closes form prematurely  
      - Network or webhook delivery issues  
    - **Sticky Note Reference:**  
      > "Step 1 - Form Trigger: Collects three user inputs... ensures structured, predictable input."

---

#### 2.2 Prompt Generation

- **Overview:**  
  This block synthesizes a precise, structured JSON prompt for the image generation model. It leverages GPT-5 chat to convert raw user inputs into a detailed prompt that controls style, composition, color palette, lighting, texture, and constraints for the icon image.

- **Nodes Involved:**  
  - GPT-5 model  
  - Generate optimized icon prompt

- **Node Details:**

  - **Node Name:** GPT-5 model  
    - **Type:** AI Language Model (OpenAI GPT-5 Chat)  
    - **Role:** Provides advanced language understanding to create structured prompts.  
    - **Configuration:**  
      - Model: "gpt-5-chat-latest"  
      - Utilizes OpenAI API credentials  
    - **Inputs:** Receives initial text prompt for prompt creation  
    - **Outputs:** Sends completion text to next node  
    - **Edge Cases & Failure Types:**  
      - API rate limits or quota exceeded  
      - Network timeouts  
      - Unexpected or ambiguous input causing poor prompt generation  
    - **Sub-workflow:** None  
    - **Sticky Note Reference:**  
      > "Step 2 - Generate optimized icon prompt: Creates a clean... precise parameters."

  - **Node Name:** Generate optimized icon prompt  
    - **Type:** Chain LLM (Langchain node)  
    - **Role:** Formats and outputs the optimized prompt as JSON for image generation.  
    - **Configuration:**  
      - Uses a template message defining a top graphic designer persona  
      - The prompt includes detailed style, composition, color palette, lighting, texture, constraints, and summary for GPT Image 1  
      - Input text dynamically composed from form data (`Icon Subject`, `Icon style`, `Icon Background`)  
    - **Key Expressions:**  
      - `"=create prompt for generateig Generate an icon of a {{ $('When form is submitted').item.json['Icon Subject'] }}, in style {{ $json['Icon style'] }} with background: {{ $json['Icon Background'] }}"`  
    - **Inputs:** User inputs from form trigger, augmented by GPT-5 model output  
    - **Outputs:** JSON prompt for image rendering  
    - **Edge Cases & Failure Types:**  
      - Expression evaluation errors  
      - Incomplete or malformed input data  
      - GPT-5 output not matching expected structure  
    - **Sticky Note Reference:**  
      > "Step 2 - Generate optimized icon prompt: Creates a clean, structured JSON prompt..."

---

#### 2.3 AI Image Rendering

- **Overview:**  
  This block uses the GPT Image 1 model to generate the actual icon image based on the structured prompt. It outputs a 400×400 PNG with transparent background encoded as base64 data.

- **Nodes Involved:**  
  - Render icon image

- **Node Details:**

  - **Node Name:** Render icon image  
    - **Type:** OpenAI Image Generation (GPT Image 1)  
    - **Role:** Generates the icon image from optimized prompt JSON.  
    - **Configuration:**  
      - Model: "gpt-image-1"  
      - Prompt: set dynamically from previous node's JSON (`$json.text`)  
      - Output: image data in base64 format  
    - **Credentials:** OpenAI API credentials configured  
    - **Inputs:** JSON prompt from the "Generate optimized icon prompt" node  
    - **Outputs:** Image data as base64 string  
    - **Edge Cases & Failure Types:**  
      - API rate limits or quota exhaustion  
      - Timeout or network errors  
      - Image generation failure or unexpected format  
    - **Sticky Note Reference:**  
      > "Step 3 - Render icon image: Uses GPT-Image-1 to generate a 400×400 PNG ..."

---

#### 2.4 Cloud Storage Upload

- **Overview:**  
  This block uploads the generated icon PNG file to a specific Google Drive folder, providing publicly accessible preview and download links.

- **Nodes Involved:**  
  - Upload icon to Google Drive

- **Node Details:**

  - **Node Name:** Upload icon to Google Drive  
    - **Type:** Google Drive Node  
    - **Role:** Uploads the base64 encoded image as a file named after the icon subject, into a predefined Drive folder.  
    - **Configuration:**  
      - File name: dynamically set from `Icon Subject` field  
      - Drive: "My Drive" (default user Drive)  
      - Folder ID: specific folder ID for "icons" (`1zgt0yHp7nfAHhvM_RHGS-nx196LNtktP`)  
      - Input data field: uses `data` field containing base64 image data  
    - **Credentials:** Google Drive OAuth2 personal account required  
    - **Inputs:** Base64 image from the "Render icon image" node  
    - **Outputs:** Contains Google Drive metadata including `thumbnailLink` and `webContentLink`  
    - **Edge Cases & Failure Types:**  
      - Authentication errors (expired or revoked tokens)  
      - Folder ID invalid or access denied  
      - Upload failures due to file size or network issues  
    - **Sticky Note Reference:**  
      > "Step 4 - Upload to Google Drive: Uploads the generated PNG ..."

---

#### 2.5 User Feedback Display

- **Overview:**  
  This block shows a confirmation form immediately after upload, displaying a thumbnail preview of the icon, a download link, and a button to restart the process for another icon generation.

- **Nodes Involved:**  
  - Display form completion

- **Node Details:**

  - **Node Name:** Display form completion  
    - **Type:** Form Node (completion display)  
    - **Role:** Presents visual confirmation and interactive options to the user post icon generation.  
    - **Configuration:**  
      - Custom HTML content includes:  
        - Thumbnail image sourced from `thumbnailLink`  
        - Download link using `webContentLink`  
        - Link to restart the icon generation form  
      - Custom CSS applied for polished UI appearance  
    - **Inputs:** Metadata output from Google Drive node (link fields)  
    - **Outputs:** None (end node)  
    - **Edge Cases & Failure Types:**  
      - Missing or invalid links causing broken images or download failures  
      - Browser incompatibility with custom HTML or CSS  
    - **Sticky Note Reference:**  
      > "Step 5 - Display Form Completion: Shows a confirmation card with thumbnail preview, download link..."

---

### 3. Summary Table

| Node Name                 | Node Type                        | Functional Role                         | Input Node(s)                  | Output Node(s)               | Sticky Note                                               |
|---------------------------|---------------------------------|---------------------------------------|-------------------------------|-----------------------------|-----------------------------------------------------------|
| When form is submitted     | Form Trigger                    | Entry point; captures user inputs     | None                          | Generate optimized icon prompt | Step 1 - Form Trigger: Collects three user inputs...      |
| GPT-5 model               | AI Language Model (OpenAI GPT-5) | Generates structured prompt text      | None (triggered by form node) | Generate optimized icon prompt | Step 2 - Generate optimized icon prompt: Creates clean... |
| Generate optimized icon prompt | Chain LLM (Langchain)           | Creates JSON prompt for image model   | When form is submitted, GPT-5 model | Render icon image            | Step 2 - Generate optimized icon prompt: Creates clean... |
| Render icon image          | OpenAI Image Generation (GPT Image 1) | Generates icon image base64 data       | Generate optimized icon prompt | Upload icon to Google Drive  | Step 3 - Render icon image: Uses GPT-Image-1 to generate...|
| Upload icon to Google Drive| Google Drive                    | Uploads icon image to Drive folder    | Render icon image              | Display form completion      | Step 4 - Upload to Google Drive: Uploads the generated PNG |
| Display form completion    | Form Completion                | Displays confirmation and links       | Upload icon to Google Drive   | None                        | Step 5 - Display Form Completion: Shows confirmation card |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Form Trigger node:**  
   - Type: `Form Trigger`  
   - Title: "Icon Generator"  
   - Fields:  
     - Dropdown: "Icon Subject" (example placeholder: Action Copy, Banana, etc.)  
     - Dropdown: "Icon Style" with the 18 detailed style options (Flat, Outline, Filled, Material Design, Neumorphic, Skeuomorphic, Gradient, Glassmorphic, 3D/Isometric, Hand-drawn, Cartoon, Minimalistic, Pixel Art, Line + Fill, Monochrome, Cyberpunk, Retro)  
     - Text input: "Icon Background"  
   - No credentials needed  
   - Connect output to next node  

2. **Create the GPT-5 model node (AI Language Model):**  
   - Type: `Langchain Chat LLM`  
   - Model: Select "gpt-5-chat-latest"  
   - Credentials: Configure OpenAI API credentials  
   - Connect input from the form trigger node's output  

3. **Create the Generate optimized icon prompt node:**  
   - Type: `Langchain Chain LLM`  
   - Configuration:  
     - Use a message template that simulates a top graphic designer specializing in icons  
     - Output a detailed JSON prompt for GPT Image 1, including style, composition, color palette, lighting, texture, constraints, and prompt summary  
     - Use dynamic expressions to inject user input from the form trigger node:  
       ```
       Generate an icon of a {{ $('When form is submitted').item.json["Icon Subject"] }}, in style {{ $json["Icon style"] }} with background: {{ $json["Icon Background"] }}
       ```  
   - Connect input from both the form trigger and GPT-5 model nodes (depending on design, typically chaining GPT-5 output here)  
   - Connect output to next node  

4. **Create the Render icon image node:**  
   - Type: `OpenAI Image Generation`  
   - Model: "gpt-image-1"  
   - Prompt: Bind to the JSON prompt output from the previous node (`$json.text`)  
   - Credentials: OpenAI API credentials  
   - Connect input from the Generate optimized icon prompt node  

5. **Create the Upload icon to Google Drive node:**  
   - Type: `Google Drive`  
   - Credentials: Google Drive OAuth2 (personal account)  
   - Configure:  
     - File name: Set dynamically to `{{$json["Icon Subject"]}}`  
     - Drive: "My Drive"  
     - Folder ID: Set to your target folder (e.g., `1zgt0yHp7nfAHhvM_RHGS-nx196LNtktP`)  
     - Input data field: map to the base64 image data from previous node (typically field named `data`)  
   - Connect input from Render icon image node  

6. **Create the Display form completion node:**  
   - Type: `Form` (completion mode)  
   - Configure:  
     - Completion title: "Icon generated"  
     - Completion message: Custom HTML to display:  
       - Thumbnail image using `thumbnailLink` from Google Drive output  
       - Download link using `webContentLink`  
       - Link back to the initial form URL for generating another icon  
     - Add custom CSS for polished styling (optional)  
   - Connect input from Upload icon to Google Drive node  

7. **Connect all nodes in this order:**  
   `When form is submitted` → `GPT-5 model` → `Generate optimized icon prompt` → `Render icon image` → `Upload icon to Google Drive` → `Display form completion`

8. **Additional Setup:**  
   - Add valid OpenAI API credentials supporting GPT-5 Chat and GPT Image models  
   - Add Google Drive OAuth2 credentials with access to the target Drive folder  
   - Ensure the Google Drive folder ID is current and accessible  
   - Optional: Customize form appearance and completion CSS  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                            | Context or Link                                                  |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------|
| The workflow is compatible with both n8n Cloud and self-hosted instances (e.g., running on Proxmox).                                                                                     | Setup Notes sticky note                                           |
| The Google Drive folder ID (`1zgt0yHp7nfAHhvM_RHGS-nx196LNtktP`) must be updated if a different storage location is desired.                                                             | Setup Notes sticky note                                           |
| This workflow can be adapted for other integrations such as Slack or Microsoft Teams by replacing the form trigger or completion nodes accordingly.                                     | Setup Notes sticky note                                           |
| The form completion node uses custom CSS for a clean and polished user interface; CSS was trimmed for Web Application Firewall compliance but can be customized further as needed.        | Display form completion sticky note                              |
| For detailed icon style options, refer to the dropdown list in the form trigger node, which covers a wide range of popular and niche icon design trends.                                | When form is submitted node configuration                        |
| The workflow leverages OpenAI’s GPT Image 1 model for high-quality, transparent PNG generation suitable for UI/UX designs.                                                              | Render icon image sticky note                                    |
| The prompt generation uses a chain LLM node with a "top graphic designer" persona to maximize image generation quality and consistency.                                                | Generate optimized icon prompt node details                     |

---

**Disclaimer:** The provided text is sourced exclusively from an automated n8n workflow. This process strictly respects content policies and contains no illegal, offensive, or protected material. All data handled is legal and public.

---