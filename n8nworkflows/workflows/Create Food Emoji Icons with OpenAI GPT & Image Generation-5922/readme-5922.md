Create Food Emoji Icons with OpenAI GPT & Image Generation

https://n8nworkflows.xyz/workflows/create-food-emoji-icons-with-openai-gpt---image-generation-5922


# Create Food Emoji Icons with OpenAI GPT & Image Generation

### 1. Workflow Overview

This workflow enables users to generate customized 3D-rendered food emoji icons using OpenAI’s GPT and image generation models, then save the generated images directly to Google Drive. It targets creative professionals, designers, or hobbyists who want to quickly create styled digital food icons based on textual input.

The workflow consists of three main logical blocks:

- **1.1 Input Reception:** Collects user input through a web form specifying the food emoji to generate.
- **1.2 AI Processing:** Uses OpenAI GPT to generate a detailed JSON style description and then to create a 3D-rendered image of the food emoji based on the style.
- **1.3 Output Storage:** Uploads the generated food emoji image to a user’s Google Drive for easy access and storage.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
This block captures the user’s desired food emoji name via a form submission webhook, serving as the entry point for the workflow.

**Nodes Involved:**  
- Trigger: Food Emoji Form Submission

**Node Details:**

- **Trigger: Food Emoji Form Submission**  
  - **Type:** Form Trigger  
  - **Role:** Webhook to capture form submissions with user input.  
  - **Configuration:**  
    - Form title: “Submit a food item”  
    - One required field: “What food emoji would you like to generate?” with placeholder text “a green apple”  
    - Form description clarifies the purpose: enter a food name to generate a 400×400-pixel 3D emoji.  
  - **Input/Output:**  
    - Input: HTTP POST from form submission  
    - Output: JSON object containing the user’s input under the key matching the form field label.  
  - **Edge Cases:**  
    - Missing or empty input (form enforces required field)  
    - Invalid characters or unsupported food names (not explicitly handled, depends on AI interpretation)  
  - **Version:** 2.2  

---

#### 1.2 AI Processing

**Overview:**  
Generates a JSON style specification for the requested food emoji, then uses this style to produce a 3D-rendered image icon via OpenAI models.

**Nodes Involved:**  
- Prepare Style‑JSON Prompt  
- LLM: Generate Style‑JSON  
- Image‑Gen: Render Food Emoji Icon

**Node Details:**

- **Prepare Style‑JSON Prompt**  
  - **Type:** Set node  
  - **Role:** Constructs a detailed prompt string embedding the user’s food input, instructing the LLM to generate a structured JSON describing the style of the emoji icon.  
  - **Configuration:**  
    - Sets a single string variable `json_generator` containing instructions requesting a JSON with sections such as form, lighting, texture, background, color_handling, and color_palette.  
    - The prompt asks for a modern, playful, semi-realistic style, 400x400 pixels, transparent background.  
  - **Input/Output:**  
    - Input: JSON from trigger node containing user input  
    - Output: JSON with `json_generator` string ready for LLM processing  
  - **Edge Cases:**  
    - Incorrect field names or missing input will cause an incomplete prompt.  
  - **Version:** 3.4  

- **LLM: Generate Style‑JSON**  
  - **Type:** Langchain OpenAI node  
  - **Role:** Calls OpenAI GPT-4.1-mini to generate a JSON object describing how to style the emoji based on the prompt.  
  - **Configuration:**  
    - Model: GPT-4.1-mini  
    - Input: Uses the `json_generator` prompt string directly from the previous node  
    - Output: JSON parsed from the LLM response (`jsonOutput` enabled)  
  - **Input/Output:**  
    - Input: Prompt string from Set node  
    - Output: JSON object describing style details  
  - **Credentials:** OpenAI API key (configured)  
  - **Edge Cases:**  
    - API quota limits or authorization errors  
    - Unexpected or malformed JSON response (parsing failure)  
    - Network timeouts or rate limits from OpenAI  
  - **Version:** 1.8  

- **Image‑Gen: Render Food Emoji Icon**  
  - **Type:** Langchain OpenAI node  
  - **Role:** Uses OpenAI’s image generation capability (model “gpt-image-1”) to create a 3D emoji icon image based on the user’s food input and the detailed style JSON from the previous node.  
  - **Configuration:**  
    - Model: gpt-image-1 (image generation)  
    - Prompt: Combines the food item input and the JSON style specification serialized as a string.  
    - Instructions emphasize a centered 400x400 px image, transparent background, no props or text, high-quality, playful and polished style consistent with mobile icons.  
  - **Input/Output:**  
    - Input: Style JSON plus user’s food item name  
    - Output: Image data in base64 or URL format (depends on OpenAI response)  
  - **Credentials:** OpenAI API key (configured)  
  - **Edge Cases:**  
    - Image generation failures, e.g., model not available  
    - API errors or exceeding rate limits  
    - Improper prompt formatting causing poor image quality  
  - **Version:** 1.8  

---

#### 1.3 Output Storage

**Overview:**  
Uploads the generated image file to Google Drive under a filename matching the user’s food item name.

**Nodes Involved:**  
- Save to Google Drive

**Node Details:**

- **Save to Google Drive**  
  - **Type:** Google Drive node  
  - **Role:** Uploads the image output from the previous node to the user’s Google Drive root folder, naming the file after the requested food emoji.  
  - **Configuration:**  
    - File name: dynamically set to the user’s food input string  
    - Drive: “My Drive” (default user drive)  
    - Folder: root folder ("/")  
    - Input data field: set to receive image data from previous node output field `data` (assumed image data)  
  - **Credentials:** Google Drive OAuth2 configured and authorized  
  - **Input/Output:**  
    - Input: Image data from Image Generation node  
    - Output: Metadata about uploaded file (file ID, URL, etc.)  
  - **Edge Cases:**  
    - Authorization errors (expired or missing OAuth tokens)  
    - Insufficient Google Drive storage space  
    - Invalid filenames or unsupported file formats  
  - **Version:** 3  

---

### 3. Summary Table

| Node Name                         | Node Type                      | Functional Role                      | Input Node(s)                         | Output Node(s)                        | Sticky Note                                                                                           |
|----------------------------------|--------------------------------|------------------------------------|-------------------------------------|-------------------------------------|----------------------------------------------------------------------------------------------------|
| Trigger: Food Emoji Form Submission | Form Trigger                   | Receive food emoji input via form  | -                                   | Prepare Style‑JSON Prompt            | ## | INPUT: Intake Form                                                                              |
| Prepare Style‑JSON Prompt         | Set                            | Build detailed style JSON prompt   | Trigger: Food Emoji Form Submission | LLM: Generate Style‑JSON             | ## | Step 1: Generate Image                                                                         |
| LLM: Generate Style‑JSON          | Langchain OpenAI (GPT-4.1-mini) | Generate style JSON for emoji      | Prepare Style‑JSON Prompt            | Image‑Gen: Render Food Emoji Icon    | ## | Step 1: Generate Image                                                                         |
| Image‑Gen: Render Food Emoji Icon | Langchain OpenAI (Image Gen)   | Generate 3D emoji-style image icon | LLM: Generate Style‑JSON             | Save to Google Drive                 | ## | Step 1: Generate Image                                                                         |
| Save to Google Drive              | Google Drive                   | Upload generated image to Drive    | Image‑Gen: Render Food Emoji Icon   | -                                   | ## | Step 2: Upload to Google Drive                                                                |
| Sticky Note                      | Sticky Note                    | Visual label for input block       | -                                   | -                                   | ## | INPUT: Intake Form                                                                             |
| Sticky Note1                     | Sticky Note                    | Visual label for generation block  | -                                   | -                                   | ## | Step 1: Generate Image                                                                         |
| Sticky Note2                     | Sticky Note                    | Visual label for upload block      | -                                   | -                                   | ## | Step 2: Upload to Google Drive                                                                |
| Sticky Note3                     | Sticky Note                    | Setup instructions and notes       | -                                   | -                                   | ## 🚀 Setup Requirements\n\nTo get started with this workflow, follow these steps:\n\n1. **🔑 Configure Credentials**: Set up your API credentials for OpenAI and Google Drive\n2. **💳 Add OpoenAI Credit**: Make sure to add credit to your OpenAI account, verify your organization (required for generating images)\n3. **📊 Connect Google Drive**: Authenticate your Google Drive account\n4. **⚙️ (Optional) Customize Prompts**: Adjust the prompts within the workflow to better suit your specific needs\n\n**Note: Each image generation will cost you about $0.17** |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger Node**  
   - Type: Form Trigger  
   - Name: “Trigger: Food Emoji Form Submission”  
   - Configure webhook with unique ID or leave default  
   - Form Title: “Submit a food item”  
   - Add one required form field: Label: “What food emoji would you like to generate?” with placeholder “a green apple”  
   - Add form description: “Enter a food name (e.g. avocado, donut) to generate a 400×400-pixel 3D emoji 🥑”  
   - Save and activate webhook  

2. **Add a Set Node to Prepare the JSON Prompt**  
   - Name: “Prepare Style‑JSON Prompt”  
   - In Parameters → Assignments, add one string variable named `json_generator`  
   - Set value to a multiline string:  
     ```
     Given the food item: "{{ $json['What food emoji would you like to generate?'] }}", generate a JSON object describing how it should be styled as a 3D-rendered emoji-style icon suitable for use in a digital food icon set. The style should be modern, playful, and semi-realistic, with a transparent background and a 400x400 pixel size.

     The JSON should include these sections:

     - form (shape, outline, detail)

     - lighting (gloss, shadow, detail)

     - texture (surface, detail)

     - background (type, detail)

     - color_handling (strategy, look, detail)

     - color_palette (detail)

     Adapt each parameter thoughtfully based on the physical properties and personality of the given food item.
     ```
   - Connect the output of the Form Trigger node to this Set node  

3. **Add an OpenAI Langchain Node to Generate Style JSON**  
   - Name: “LLM: Generate Style‑JSON”  
   - Select the OpenAI credential configured in your n8n instance  
   - Model: Choose “gpt-4.1-mini”  
   - Enable JSON output parsing (`jsonOutput: true`)  
   - Input messages: Single system or user message with content set to `={{ $json.json_generator }}` from previous Set node  
   - Connect this node’s input from the Set node  

4. **Add an OpenAI Langchain Node for Image Generation**  
   - Name: “Image‑Gen: Render Food Emoji Icon”  
   - Same OpenAI credential as before  
   - Model: “gpt-image-1”  
   - Set resource type: Image generation  
   - Prompt:  
     ```
     Generate a 3D-rendered emoji-style digital icon of a {{ $('Trigger: Food Emoji Form Submission').item.json['What food emoji would you like to generate?'] }}, designed with the following visual specifications:
     {{ $json.message.content.toJsonString() }}

     Render the icon centered in a 400x400 pixel square, isolated on a transparent background, with no props or text. The result should look like a high-quality digital food emoji: slightly exaggerated, clean, friendly, and polished — consistent with a modern mobile icon set.
     ```
   - Connect input from “LLM: Generate Style‑JSON” node  

5. **Add Google Drive Node to Save the Image**  
   - Name: “Save to Google Drive”  
   - Authenticate with Google Drive OAuth2 credentials  
   - Drive: “My Drive”  
   - Folder: Root ("/")  
   - File Name: `={{ $('Trigger: Food Emoji Form Submission').item.json['What food emoji would you like to generate?'] }}` (dynamic naming)  
   - Input Data Field Name: “data” (ensure this matches the image output field from the previous node)  
   - Connect input from “Image‑Gen: Render Food Emoji Icon” node  

6. **Add Sticky Notes (Optional for clarity)**  
   - Create sticky notes with the following content for visual grouping and instructions:  
     - Input block: “## | INPUT: Intake Form” near the Form Trigger node  
     - Step 1: “## | Step 1: Generate Image” near the AI and image generation nodes  
     - Step 2: “## | Step 2: Upload to Google Drive” near the Google Drive node  
     - Setup instructions with notes about credential setup and cost  

7. **Activate the Workflow**  
   - Ensure all nodes have the correct credentials assigned  
   - Test the workflow by submitting the form and verifying the generated image appears in Google Drive  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                          | Context or Link                                                                              |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------|
| To start, configure OpenAI and Google Drive credentials properly and ensure billing and API quotas are sufficient. Each image generation costs approximately $0.17.                                                                                                                                                | Sticky Note3 content                                                                         |
| The workflow uses OpenAI GPT-4.1-mini for generating JSON style descriptions and the “gpt-image-1” model for image generation, which requires an OpenAI account with image generation permissions enabled.                                                                                                           | Node configuration details                                                                   |
| Form submissions are received via webhook; to use, activate the workflow and access the provided webhook URL to submit food emoji requests programmatically or via the embedded form.                                                                                                                                | Trigger node explanation                                                                     |
| The style JSON is designed for flexibility—users can customize the prompt to alter the style attributes such as lighting, texture, and color palette to produce different artistic effects.                                                                                                                         | Prompt content in Set node                                                                   |
| For reference and additional examples of OpenAI n8n integrations, visit https://n8n.io/integrations/n8n-nodes-langchain.openAi                                                                                                                                                                                     | n8n OpenAI node integration page                                                            |
| Generated images are saved to the root of the user’s Google Drive by default; this can be changed by modifying the folderId parameter in the Google Drive node.                                                                                                                                                      | Google Drive node configuration                                                             |

---

**Disclaimer:** The provided text is exclusively derived from an automated workflow created with n8n, a tool for integration and automation. This processing strictly adheres to applicable content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.