Generate Multiple AI Images with DALL·E 2 and Upload to Google Drive

https://n8nworkflows.xyz/workflows/generate-multiple-ai-images-with-dall-e-2-and-upload-to-google-drive-7222


# Generate Multiple AI Images with DALL·E 2 and Upload to Google Drive

### 1. Workflow Overview

This workflow automates the generation of multiple AI images using OpenAI’s DALL·E 2 model based on a single textual prompt, and uploads each generated image to a specified Google Drive folder. It is designed for use cases where multiple variations of an image are desired from one prompt, facilitating batch image creation and storage in cloud drive automatically.

The workflow is logically divided into the following blocks:

- **1.1 Input Initialization:** Manual trigger to start and setting the base prompt and filename.
- **1.2 Duplication of Prompts:** Creating multiple copies of the prompt to generate several image variations.
- **1.3 Batch Processing:** Looping over each prompt variation sequentially.
- **1.4 AI Image Generation:** Using OpenAI’s DALL·E 2 model to generate images from prompts.
- **1.5 Upload to Google Drive:** Saving each generated image to a designated Google Drive folder with a unique name.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Initialization

- **Overview:** This block initiates the workflow manually and sets the main parameters: the image description prompt and base filename.
- **Nodes Involved:**  
  - Start Workflow  
  - Set Image Prompt

- **Node Details:**

  - **Start Workflow**  
    - Type: Manual Trigger  
    - Role: Allows user to manually start the workflow for testing or on demand.  
    - Configuration: Default manual trigger without parameters.  
    - Inputs: None  
    - Outputs: Trigger event to next node.  
    - Edge Cases: None expected; manual trigger ensures control over execution.

  - **Set Image Prompt**  
    - Type: Set  
    - Role: Stores the textual prompt and the base name for generated images.  
    - Configuration: Two fields assigned:  
      - `Prompt` (string): e.g., "Make an image of an attractive woman standing in New York City"  
      - `Name` (string): e.g., "woman-nyc"  
    - Inputs: Receives trigger from manual start  
    - Outputs: Passes JSON with `Prompt` and `Name` fields forward  
    - Edge Cases: Ensure prompt text is non-empty; empty prompts may cause generation errors.

#### 1.2 Duplication of Prompts

- **Overview:** Creates multiple copies of the input prompt to generate several image variations.  
- **Nodes Involved:**  
  - Duplicate Rows

- **Node Details:**

  - **Duplicate Rows**  
    - Type: Code (JavaScript)  
    - Role: Generates three identical prompt items with a distinct `run` index (1, 2, 3) to differentiate each variation.  
    - Configuration: Custom JS code:  
      ```javascript
      const original = items[0].json;
      return [
        { json: { ...original, run: 1 } },
        { json: { ...original, run: 2 } },
        { json: { ...original, run: 3 } },
      ];
      ```  
    - Inputs: Receives JSON with `Prompt` and `Name`  
    - Outputs: Produces an array of 3 items, each with `run` property added  
    - Edge Cases: Assumes single input item; multiple inputs would require adapting code.  
    - Failure Modes: JS syntax errors or empty input will cause failure.

#### 1.3 Batch Processing

- **Overview:** Processes each prompt variation sequentially, ensuring image generation and upload happen one by one.  
- **Nodes Involved:**  
  - Loop Over Items

- **Node Details:**

  - **Loop Over Items**  
    - Type: Split In Batches  
    - Role: Splits the input array of prompt variations into batches of size 1 for sequential processing.  
    - Configuration: Batch size = 1; reset option disabled (default)  
    - Inputs: Receives array of prompt variations (3 items) from Duplicate Rows  
    - Outputs: Passes one item at a time downstream  
    - Edge Cases: Handles empty input arrays gracefully, resulting in no execution downstream.

#### 1.4 AI Image Generation

- **Overview:** Calls OpenAI’s DALL·E 2 image generation API to create an image based on the current prompt.  
- **Nodes Involved:**  
  - Generate an image

- **Node Details:**

  - **Generate an image**  
    - Type: OpenAI (LangChain) Node  
    - Role: Sends the prompt to OpenAI’s DALL·E 2 model and retrieves an AI-generated image.  
    - Configuration:  
      - Model: `dall-e-2`  
      - Prompt: Expression `={{ $json.Prompt }}` (uses current batch item prompt)  
      - Options: Default (no additional image size or style specified)  
    - Credentials: OpenAI API key (configured in n8n credentials)  
    - Inputs: Single prompt JSON from batch  
    - Outputs: JSON including image data URL or image URL, depending on API response  
    - Edge Cases:  
      - API key invalid or quota exceeded → authentication errors  
      - Network timeouts or API downtime → retry or fail  
      - Invalid prompt or empty string → generation failure  
    - Version: n8n node version 1.8

#### 1.5 Upload to Google Drive

- **Overview:** Uploads the generated image file to a specified Google Drive folder with a unique filename.  
- **Nodes Involved:**  
  - Upload to Google Drive

- **Node Details:**

  - **Upload to Google Drive**  
    - Type: Google Drive Node  
    - Role: Saves the generated image from OpenAI to Google Drive with a dynamic filename.  
    - Configuration:  
      - File Name: Expression combining base name and run number:  
        `={{ $('Set Image Prompt').item.json.Name }} - {{ $('Duplicate Rows').item.json.run }}`  
      - Drive ID: "My Drive"  
      - Folder ID: Specific folder ID (`1TnDibwPPPUm3VbmETiqWDVhtaUTLJ6mn`) where images are stored  
      - Options: Default (no additional advanced options)  
    - Credentials: Google Drive OAuth2 (configured in n8n credentials)  
    - Inputs: Receives image data from Generate an image node  
    - Outputs: Upload confirmation data (file ID, metadata)  
    - Edge Cases:  
      - OAuth token expiration or invalid credentials → authentication errors  
      - Folder permissions insufficient → upload failure  
      - Large image files exceeding Google Drive limits → errors  
    - Version: n8n node version 3

---

### 3. Summary Table

| Node Name          | Node Type                       | Functional Role                          | Input Node(s)            | Output Node(s)                | Sticky Note                                                                                                          |
|--------------------|--------------------------------|----------------------------------------|--------------------------|------------------------------|----------------------------------------------------------------------------------------------------------------------|
| Start Workflow     | Manual Trigger                 | Starts the workflow manually           | —                        | Set Image Prompt             | See overall step-by-step setup instructions in Sticky Note (detailed guide)                                          |
| Set Image Prompt    | Set                            | Stores prompt text and base filename   | Start Workflow           | Duplicate Rows               | See overall step-by-step setup instructions in Sticky Note                                                          |
| Duplicate Rows      | Code (JavaScript)              | Creates multiple prompt variations      | Set Image Prompt          | Loop Over Items              | See overall step-by-step setup instructions in Sticky Note                                                          |
| Loop Over Items     | Split In Batches               | Processes prompts one at a time         | Duplicate Rows            | Generate an image, Upload to Google Drive | See overall step-by-step setup instructions in Sticky Note                                                          |
| Generate an image   | OpenAI (LangChain)             | Generates AI image with DALL·E 2        | Loop Over Items           | Loop Over Items              | See overall step-by-step setup instructions in Sticky Note                                                          |
| Upload to Google Drive | Google Drive                  | Uploads generated image to Drive        | Generate an image         | Loop Over Items              | See overall step-by-step setup instructions in Sticky Note                                                          |
| Sticky Note        | Sticky Note                   | Provides detailed setup and usage guide | —                        | —                            | Contains comprehensive step-by-step setup, API key instructions, workflow overview, customization tips, contacts     |
| Sticky Note1       | Sticky Note                   | Provides detailed build steps           | —                        | —                            | Step-by-step node setup and configuration instructions                                                                |
| Sticky Note2       | Sticky Note                   | Contact and support information          | —                        | —                            | **Need more help?** Website: https://ynteractive.com Email: robert@ynteractive.com                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Add node: *Manual Trigger*  
   - No parameters needed  
   - This node will start the workflow manually.

2. **Create Set Node for Prompt**  
   - Add node: *Set*  
   - Add two fields:  
     - `Prompt`: string; example value: "Make an image of an attractive woman standing in New York City"  
     - `Name`: string; example value: "woman-nyc"  
   - Connect *Manual Trigger* → *Set*

3. **Create Code Node to Duplicate Rows**  
   - Add node: *Code* (JavaScript)  
   - Paste the following JS code:  
     ```javascript
     const original = items[0].json;
     return [
       { json: { ...original, run: 1 } },
       { json: { ...original, run: 2 } },
       { json: { ...original, run: 3 } },
     ];
     ```  
   - Connect *Set* → *Code*

4. **Add Split In Batches Node**  
   - Add node: *Split In Batches*  
   - Set batch size to `1`  
   - Connect *Code* → *Split In Batches*

5. **Add OpenAI Image Generation Node**  
   - Add node: *OpenAI (LangChain)*  
   - Set model to `dall-e-2`  
   - Set prompt to expression: `={{ $json.Prompt }}`  
   - Leave options default unless customization needed  
   - Set credentials to configured OpenAI API key in n8n  
   - Connect *Split In Batches* → *OpenAI Image Generation*

6. **Add Google Drive Node**  
   - Add node: *Google Drive*  
   - Set file name to expression:  
     `={{ $('Set Image Prompt').item.json.Name }} - {{ $('Duplicate Rows').item.json.run }}`  
   - Select drive as “My Drive”  
   - Enter or select folder ID where images should be saved  
   - Connect to Google Drive OAuth2 credentials in n8n  
   - Connect *OpenAI Image Generation* → *Google Drive*

7. **Connect Google Drive Node Output Back to Split In Batches**  
   - This ensures loop continues for all batches:  
   - Connect *Google Drive* node output → *Split In Batches* node input (second output path)

8. **Final Connections and Testing**  
   - Confirm full flow:  
     Manual Trigger → Set → Duplicate Rows → Split In Batches → (Generate Image → Upload to Google Drive) → Loop back to Split In Batches until done  
   - Ensure credentials are valid and tested  
   - Run workflow manually and verify images upload to Drive.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                             | Context or Link                                      |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------|
| Step-by-step setup instructions, including API key creation for OpenAI and Google Drive, node-by-node configuration, and workflow execution details.                                                                                                     | Provided as sticky notes inside the workflow JSON   |
| Customization tips include modifying the number of prompt variations in the code node or dynamically setting prompts via external data sources like Google Sheets or Webhooks.                                                                            | Sticky notes in workflow                             |
| Contact for support and additional help: Website https://ynteractive.com and Email robert@ynteractive.com                                                                                                                                                | Sticky Note2 node                                    |
| OpenAI Platform for API keys: https://platform.openai.com/                                                                                                                                                                                               | Setup instructions                                   |
| Google Cloud Console for Google Drive API and OAuth2 credentials setup: https://console.cloud.google.com/                                                                                                                                                 | Setup instructions                                   |

---

**Disclaimer:** The content described is derived exclusively from an automated n8n workflow, compliant with content policies and handling only legal and public data.