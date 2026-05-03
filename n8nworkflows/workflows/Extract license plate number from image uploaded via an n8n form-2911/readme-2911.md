Extract license plate number from image uploaded via an n8n form

https://n8nworkflows.xyz/workflows/extract-license-plate-number-from-image-uploaded-via-an-n8n-form-2911


# Extract license plate number from image uploaded via an n8n form

### 1. Workflow Overview

This workflow demonstrates how to extract a license plate number from an image of a car uploaded via an n8n form. It serves as a simple example of integrating file upload triggers with large language models (LLMs) for image-to-text analysis. The workflow is designed for quick prototyping and experimentation with different LLM models and prompts, specifically using OpenRouter.ai as the LLM provider.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Captures the image file uploaded by the user through an n8n form trigger.
- **1.2 Configuration Setup:** Sets the LLM model and prompt parameters dynamically for the image-to-text extraction task.
- **1.3 AI Processing:** Sends the image and prompt to an LLM via OpenRouter for license plate extraction.
- **1.4 Output Presentation:** Displays the extracted license plate number back to the user on a form completion page.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
This block handles user input by providing a web form where users can upload an image file. The uploaded image is then passed downstream for processing.

**Nodes Involved:**  
- FromTrigger

**Node Details:**

- **FromTrigger**  
  - Type: Form Trigger  
  - Role: Entry point for the workflow; receives image file uploads via a web form.  
  - Configuration:  
    - Form title: "Analyse image"  
    - Form description: "To analyse an image, upload it here."  
    - Form field: Single required file input accepting `.jpg` and `.png` files labeled "Image".  
    - Response mode: "lastNode" (returns data from the last executed node after workflow completion).  
  - Input: External HTTP request (form submission)  
  - Output: JSON containing uploaded file data (binary image)  
  - Edge cases / Potential failures:  
    - User submits no file or unsupported file type (form validation prevents this).  
    - Network or webhook errors during submission.  
  - Version: 2.2  

---

#### 1.2 Configuration Setup

**Overview:**  
This block sets the parameters for the LLM processing, including the choice of model and the prompt text that instructs the LLM on what to extract from the image.

**Nodes Involved:**  
- Settings

**Node Details:**

- **Settings**  
  - Type: Set node  
  - Role: Defines workflow variables for the LLM model and prompt text.  
  - Configuration:  
    - Assigns two string variables:  
      - `model`: `"openai/gpt-4o"` (default LLM model used)  
      - `prompt`: `"Extract the number of the license plate on the front-most car depicted in the attached image and return only the extracted characters without any other text or structure."`  
    - Includes other fields from input unchanged.  
  - Input: JSON from FromTrigger node (image upload data)  
  - Output: JSON enriched with `model` and `prompt` variables  
  - Edge cases / Potential failures:  
    - Incorrect or unsupported model string could cause downstream LLM call failures.  
    - Prompt text syntax errors unlikely but could affect LLM output quality.  
  - Version: 3.4  

---

#### 1.3 AI Processing

**Overview:**  
This block sends the image and prompt to the OpenRouter LLM for processing. It uses a LangChain node to format the prompt and an OpenRouter LLM node to perform the actual AI inference.

**Nodes Involved:**  
- Basic LLM Chain  
- OpenRouter LLM

**Node Details:**

- **Basic LLM Chain**  
  - Type: LangChain Chain LLM node  
  - Role: Prepares and sends the prompt and image data to the LLM for text extraction.  
  - Configuration:  
    - Text input: Uses expression `{{$json.prompt}}` to dynamically insert the prompt from the Settings node.  
    - Message template: Human message with type `imageBinary`, referencing the binary image data key `"Image"`.  
    - Prompt type: Defined prompt (not free text).  
  - Input: JSON with prompt and binary image data from Settings node and OpenRouter LLM node respectively.  
  - Output: JSON containing the LLM response text (extracted license plate number).  
  - Edge cases / Potential failures:  
    - Missing or corrupted image binary data.  
    - Expression evaluation errors if prompt variable missing.  
    - LLM response timeouts or errors.  
  - Version: 1.5  

- **OpenRouter LLM**  
  - Type: LangChain OpenRouter LLM node  
  - Role: Executes the LLM inference call to OpenRouter API.  
  - Configuration:  
    - Model: Dynamic, set via expression `{{$json.model}}` from Settings node.  
    - Options: Default (empty).  
    - Credentials: Uses OpenRouter API key credential (must be configured with user’s API key).  
  - Input: Receives prompt and image data from Basic LLM Chain node.  
  - Output: LLM response passed back to Basic LLM Chain node.  
  - Edge cases / Potential failures:  
    - Authentication errors if API key invalid or missing.  
    - API rate limits or quota exceeded.  
    - Network connectivity issues.  
  - Version: 1  

---

#### 1.4 Output Presentation

**Overview:**  
This block presents the extracted license plate number to the user after processing is complete.

**Nodes Involved:**  
- FormResultPage

**Node Details:**

- **FormResultPage**  
  - Type: Form node (completion page)  
  - Role: Displays the extracted license plate number as a completion message on the form page.  
  - Configuration:  
    - Completion title: "Extracted information:"  
    - Completion message: Uses expression `{{$json.text}}` to show the extracted license plate number from the LLM output.  
  - Input: Receives LLM output from Basic LLM Chain node.  
  - Output: HTTP response to user with completion message.  
  - Edge cases / Potential failures:  
    - Missing or empty LLM output text.  
    - Rendering errors on the form page.  
  - Version: 1  

---

### 3. Summary Table

| Node Name        | Node Type                          | Functional Role                  | Input Node(s)       | Output Node(s)        | Sticky Note                                                                                                         |
|------------------|----------------------------------|--------------------------------|---------------------|-----------------------|---------------------------------------------------------------------------------------------------------------------|
| FromTrigger      | Form Trigger                     | Receives image upload via form  | (external trigger)  | Settings              |                                                                                                                     |
| Settings         | Set                             | Sets LLM model and prompt       | FromTrigger         | Basic LLM Chain       |                                                                                                                     |
| OpenRouter LLM   | LangChain OpenRouter LLM        | Calls OpenRouter API for LLM    | Basic LLM Chain     | Basic LLM Chain       | Requires OpenRouter API key credential configured with your individual API key.                                     |
| Basic LLM Chain  | LangChain Chain LLM             | Sends prompt and image to LLM   | Settings, OpenRouter LLM | FormResultPage     | Uses dynamic prompt and image binary data to extract license plate number.                                          |
| FormResultPage   | Form (completion page)           | Displays extracted license plate| Basic LLM Chain     | (HTTP response)       | Shows extracted license plate number as completion message.                                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger node ("FromTrigger"):**  
   - Set form title to "Analyse image".  
   - Add a required file upload field labeled "Image" accepting `.jpg` and `.png` files.  
   - Set response mode to "lastNode".  
   - Save webhook URL for external access.

2. **Add a Set node ("Settings"):**  
   - Connect input from "FromTrigger".  
   - Add two string fields:  
     - `model` with value `"openai/gpt-4o"` (default LLM model).  
     - `prompt` with value `"Extract the number of the license plate on the front-most car depicted in the attached image and return only the extracted characters without any other text or structure."`  
   - Enable "Include Other Fields" to pass through the image data.

3. **Add an OpenRouter LLM node ("OpenRouter LLM"):**  
   - Connect input from "Basic LLM Chain" (to be created next).  
   - Set model parameter to expression `{{$json.model}}` to use the model from "Settings".  
   - Leave options empty.  
   - Configure credentials with your OpenRouter API key (create credential if not existing).

4. **Add a LangChain Chain LLM node ("Basic LLM Chain"):**  
   - Connect input from "Settings" node.  
   - Set text input to expression `{{$json.prompt}}`.  
   - Configure messages with one HumanMessagePromptTemplate of type `imageBinary` referencing binary image data key `"Image"`.  
   - Set prompt type to "define".  
   - Connect the "OpenRouter LLM" node as the AI language model provider for this node.

5. **Connect "Basic LLM Chain" output to a Form node ("FormResultPage"):**  
   - Set operation to "completion".  
   - Set completion title to "Extracted information:".  
   - Set completion message to expression `{{$json.text}}` to display the extracted license plate number.

6. **Set node connections:**  
   - FromTrigger → Settings → Basic LLM Chain → FormResultPage  
   - Basic LLM Chain → OpenRouter LLM (as AI language model)  

7. **Activate the workflow and test:**  
   - Submit an image of a car with a visible license plate via the form URL.  
   - The workflow extracts and displays the license plate number.

---

### 5. General Notes & Resources

| Note Content                                                                                                                        | Context or Link                                                                                 |
|------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| This workflow uses OpenRouter.ai for LLM inference; ensure you have an account, API key, and credits.                              | https://openrouter.ai                                                                           |
| You can adapt the prompt in the "Settings" node to other image-to-text tasks like summarization, location detection, or OCR.      | See "How to adapt" section in workflow description.                                            |
| Recommended LLM models for this demo: google/gemini-2.0-flash-001, meta-llama/llama-3.2-90b-vision-instruct, openai/gpt-4o.       |                                                                                               |
| For production-grade image text extraction, consider specialized APIs like Google Cloud Vision, Microsoft Azure Computer Vision.  |                                                                                               |
| The workflow demonstrates a simple prototype and is not optimized for error handling or high throughput.                          |                                                                                               |

---

This documentation fully describes the workflow’s structure, node configurations, data flow, and setup instructions, enabling reproduction, modification, and troubleshooting.