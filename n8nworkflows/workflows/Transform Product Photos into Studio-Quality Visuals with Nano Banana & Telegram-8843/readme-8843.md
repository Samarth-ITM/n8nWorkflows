Transform Product Photos into Studio-Quality Visuals with Nano Banana & Telegram

https://n8nworkflows.xyz/workflows/transform-product-photos-into-studio-quality-visuals-with-nano-banana---telegram-8843


# Transform Product Photos into Studio-Quality Visuals with Nano Banana & Telegram

---
### 1. Workflow Overview

This workflow automates the transformation of product photos into studio-quality visuals using the Nano Banana image generation API, integrated within a Telegram messaging context. It accepts product image submissions via a form, processes images through AI-powered enhancement and generation, and delivers the refined visuals back to users on Telegram.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Handles form submissions containing product images and background images.
- **1.2 AI Prompt Generation:** Uses OpenAI chat and LangChain nodes to generate prompts for image transformation based on inputs.
- **1.3 Image Upload and Preparation:** Uploads original product and background images to external services preparing them for processing.
- **1.4 Batch Processing & Image Generation:** Processes each image in batches, calls Nano Banana API to generate enhanced images, waits for processing completion, and retrieves the results.
- **1.5 Result Handling & Delivery:** Downloads the final images and sends them to users via Telegram.
- **1.6 Error Handling & Loop Control:** Manages failed attempts and retries image generation when necessary.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** Captures product photo submissions and background images through a form trigger to initiate processing.
- **Nodes Involved:** `Form Submission`
- **Node Details:**
  - **Form Submission**
    - Type: `formTrigger`
    - Role: Entry point node triggered by form input containing product and background images.
    - Config: Configured with a webhook ID to receive external form data.
    - Inputs: External HTTP form submission.
    - Outputs: Passes data to image upload nodes and prompt generation.
    - Edge Cases: Missing or malformed form data; webhook connectivity issues.

#### 2.2 AI Prompt Generation

- **Overview:** Uses an OpenAI chat model and LangChain agents to generate structured prompts for the Nano Banana image generation API based on the submitted images.
- **Nodes Involved:** `OpenAI Chat Model2`, `Prompt Generator1`, `Structured Output Parser1`
- **Node Details:**
  - **OpenAI Chat Model2**
    - Type: `lmChatOpenAi`
    - Role: Generates conversational AI outputs forming the basis of image transformation prompts.
    - Config: Uses an OpenAI credential; configured with appropriate model parameters.
    - Inputs: Connected from `Prompt Generator1` node as AI language model input.
    - Outputs: Passes AI-generated prompt text to `Prompt Generator1`.
    - Edge Cases: API rate limits, authentication errors, large prompt size issues.
  - **Prompt Generator1**
    - Type: `agent`
    - Role: Acts as an orchestrator to generate precise prompt instructions for the image generation.
    - Config: Uses AI output parser; linked to `Structured Output Parser1`.
    - Inputs: Receives parsed AI output.
    - Outputs: Passes prompt data to the merging node.
    - Edge Cases: Parsing errors, incomplete prompt generation.
  - **Structured Output Parser1**
    - Type: `outputParserStructured`
    - Role: Parses AI responses into structured data usable by downstream nodes.
    - Config: Set to extract relevant fields for image generation.
    - Inputs: Receives raw AI output.
    - Outputs: Sends structured data to `Prompt Generator1`.
    - Edge Cases: Parsing failures due to unexpected AI output formats.

#### 2.3 Image Upload and Preparation

- **Overview:** Uploads product and background images to external storage or processing APIs and prepares data for batch processing.
- **Nodes Involved:** `Upload Product Image`, `Upload Background1`, `Product Image`, `Background Image`, `Merge1`
- **Node Details:**
  - **Upload Product Image**
    - Type: `httpRequest`
    - Role: Uploads product images from form submission to an external service.
    - Config: Configured with API endpoint and authentication if needed.
    - Inputs: Connected from `Form Submission`.
    - Outputs: Passes uploaded image data to `Product Image`.
    - Edge Cases: Upload failures, network timeouts.
  - **Upload Background1**
    - Type: `httpRequest`
    - Role: Uploads background images similarly to product images.
    - Config: Includes error continuation (`continueErrorOutput`) to allow workflow to continue even if background upload fails.
    - Inputs: Connected from `Form Submission`.
    - Outputs: Passes data to `Background Image` node twice (duplicate output).
    - Edge Cases: Upload failures; handled gracefully.
  - **Product Image** and **Background Image**
    - Type: `set`
    - Role: Prepare and standardize the uploaded image data for merging.
    - Config: Sets variables or fields relevant for downstream processing.
    - Inputs: Receive outputs from upload nodes.
    - Outputs: Both connect to `Merge1`.
    - Edge Cases: Missing data sets.
  - **Merge1**
    - Type: `merge`
    - Role: Combines product image, background image, and AI prompt data into a single data stream.
    - Config: Merges three inputs (`Product Image`, `Background Image`, and `Prompt Generator1`).
    - Inputs: Receives from image preparation and prompt generation.
    - Outputs: Sends merged data to the `Split Out` node.
    - Edge Cases: Mismatched data lengths or missing inputs.

#### 2.4 Batch Processing & Image Generation

- **Overview:** Splits the merged input into batches, iterates over each image, calls Nano Banana API to generate studio-quality images, waits for processing, and retrieves generated images.
- **Nodes Involved:** `Split Out`, `Loop Over Items`, `Image Gen (Nano Banana)`, `Wait1`, `Get Image`, `Image created`, `failed`
- **Node Details:**
  - **Split Out**
    - Type: `splitOut`
    - Role: Splits merged data into individual items for batch processing.
    - Inputs: From `Merge1`.
    - Outputs: To `Loop Over Items`.
    - Edge Cases: Empty input arrays.
  - **Loop Over Items**
    - Type: `splitInBatches`
    - Role: Iterates over each image item in batches.
    - Inputs: From `Split Out`.
    - Outputs: Two outputs, one to `Download Images` and another to `Image Gen (Nano Banana)` for generation.
    - Edge Cases: Batch size misconfigurations.
  - **Image Gen (Nano Banana)**
    - Type: `httpRequest`
    - Role: Calls the Nano Banana API to generate enhanced images based on prompts and uploaded images.
    - Config: Configured with API endpoint, headers, and authentication as per Nano Banana API specs.
    - Inputs: From `Loop Over Items`.
    - Outputs: To `Wait1`.
    - Edge Cases: API failures, authentication errors, rate limits.
  - **Wait1**
    - Type: `wait`
    - Role: Pauses workflow execution allowing Nano Banana to process images asynchronously.
    - Inputs: From `Image Gen (Nano Banana)`.
    - Outputs: To `Get Image`.
    - Edge Cases: Timeout or excessive wait durations.
  - **Get Image**
    - Type: `httpRequest`
    - Role: Polls or fetches the generated image from Nano Banana API after processing delay.
    - Inputs: From `Wait1`.
    - Outputs: To `Image created`.
    - Edge Cases: Image not ready, API errors.
  - **Image created**
    - Type: `if`
    - Role: Checks if the image was successfully created.
    - Inputs: From `Get Image`.
    - Outputs: Success path to `Loop Over Items` for next image; failure path to `failed` node.
    - Edge Cases: False negatives or incorrect success flags.
  - **failed**
    - Type: `if`
    - Role: Handles failed image generations by deciding whether to retry or skip.
    - Inputs: From `Image created`.
    - Outputs: Retry path loops back to `Loop Over Items`; fallback path to `Wait1`.
    - Edge Cases: Infinite retry loops; thresholds not defined.

#### 2.5 Result Handling & Delivery

- **Overview:** Downloads the generated images and sends them to Telegram users.
- **Nodes Involved:** `Download Images`, `Send Images`
- **Node Details:**
  - **Download Images**
    - Type: `httpRequest`
    - Role: Downloads the final generated images from Nano Banana or intermediary storage.
    - Inputs: From `Loop Over Items`.
    - Outputs: To `Send Images`.
    - Edge Cases: Download failures, broken URLs.
  - **Send Images**
    - Type: `telegram`
    - Role: Sends processed images to users via Telegram bot.
    - Config: Configured with Telegram OAuth2 credentials; sends messages or media groups.
    - Inputs: From `Download Images`.
    - Outputs: Terminal node.
    - Edge Cases: Telegram API rate limits, chat ID errors, message size limits.

#### 2.6 Notes and Miscellaneous

- **Sticky Notes:** There are several sticky notes in the workflow (`Sticky Note5`, `Sticky Note7`, `Sticky Note8`) that contain empty content and do not add operational context.
- **General:** The workflow uses webhook IDs and OAuth2 credentials which must be properly configured for form submissions and Telegram integration.

---

### 3. Summary Table

| Node Name           | Node Type                      | Functional Role                                     | Input Node(s)               | Output Node(s)             | Sticky Note                     |
|---------------------|--------------------------------|----------------------------------------------------|-----------------------------|-----------------------------|--------------------------------|
| Form Submission     | formTrigger                    | Entry point to receive product and background images | None                        | Upload Background1, Upload Product Image, Prompt Generator1 |                                |
| Upload Background1  | httpRequest                   | Uploads background image to external service       | Form Submission              | Background Image (x2)        |                                |
| Background Image    | set                           | Prepares background image data                      | Upload Background1           | Merge1                      |                                |
| Upload Product Image| httpRequest                   | Uploads product image to external service          | Form Submission              | Product Image               |                                |
| Product Image       | set                           | Prepares product image data                         | Upload Product Image         | Merge1                      |                                |
| OpenAI Chat Model2  | lmChatOpenAi                  | Generates AI chat-based prompt                      | Prompt Generator1 (ai_languageModel) | Prompt Generator1 (ai_languageModel output) |                                |
| Prompt Generator1   | agent                         | Generates structured prompt for image generation   | Structured Output Parser1    | Merge1                      |                                |
| Structured Output Parser1 | outputParserStructured     | Parses AI output into structured data               | OpenAI Chat Model2           | Prompt Generator1           |                                |
| Merge1              | merge                         | Combines product, background images and prompts    | Product Image, Background Image, Prompt Generator1 | Split Out                   |                                |
| Split Out           | splitOut                      | Splits merged data into individual items            | Merge1                      | Loop Over Items             |                                |
| Loop Over Items     | splitInBatches                | Processes each image in batches                      | Split Out                   | Download Images, Image Gen (Nano Banana) |                                |
| Download Images     | httpRequest                   | Downloads final generated images                     | Loop Over Items             | Send Images                 |                                |
| Image Gen (Nano Banana) | httpRequest                 | Calls Nano Banana API to generate enhanced images   | Loop Over Items             | Wait1                      |                                |
| Wait1               | wait                          | Waits for image generation processing completion    | Image Gen (Nano Banana)     | Get Image                  |                                |
| Get Image           | httpRequest                   | Retrieves generated image                            | Wait1                       | Image created              |                                |
| Image created       | if                            | Checks if image creation succeeded                   | Get Image                   | Loop Over Items (success), failed (failure) |                                |
| failed              | if                            | Handles failed image generation                       | Image created               | Loop Over Items (retry), Wait1 (fallback) |                                |
| Send Images         | telegram                      | Sends final images to Telegram users                 | Download Images             | None                       |                                |
| Sticky Note5        | stickyNote                    | Informational note (empty content)                   | None                       | None                       |                                |
| Sticky Note7        | stickyNote                    | Informational note (empty content)                   | None                       | None                       |                                |
| Sticky Note8        | stickyNote                    | Informational note (empty content)                   | None                       | None                       |                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create `Form Submission` node:**
   - Type: `formTrigger`
   - Configure a webhook to accept product and background image uploads.
   - Save webhook URL for form integration.

2. **Create `Upload Background1` node:**
   - Type: `httpRequest`
   - Configure with API endpoint to upload background image.
   - Set method to POST and include authentication if needed.
   - Set error handling to "Continue on Error."
   - Connect input from `Form Submission`.

3. **Create `Background Image` node:**
   - Type: `set`
   - Map necessary fields from upload response (e.g., image URL).
   - Connect input from `Upload Background1`.

4. **Create `Upload Product Image` node:**
   - Type: `httpRequest`
   - Configure with API endpoint to upload product image.
   - Connect input from `Form Submission`.

5. **Create `Product Image` node:**
   - Type: `set`
   - Map fields from product image upload response.
   - Connect input from `Upload Product Image`.

6. **Create `OpenAI Chat Model2` node:**
   - Type: `lmChatOpenAi`
   - Configure OpenAI credentials (API key).
   - Set model parameters (e.g., GPT-4 or GPT-3.5).
   - Connect input from `Prompt Generator1` AI language model output.

7. **Create `Structured Output Parser1` node:**
   - Type: `outputParserStructured`
   - Configure parsing logic to extract relevant prompts.
   - Connect input from `OpenAI Chat Model2` AI output.

8. **Create `Prompt Generator1` node:**
   - Type: `agent`
   - Configure to generate prompts using structured output.
   - Connect input from `Structured Output Parser1`.
   - Connect AI language model output to `OpenAI Chat Model2`.
   - Connect main output to `Merge1`.

9. **Create `Merge1` node:**
   - Type: `merge`
   - Configure to merge three inputs:
     - Input 1: `Background Image`
     - Input 2: `Product Image`
     - Input 3: `Prompt Generator1`
   - Connect outputs accordingly.

10. **Create `Split Out` node:**
    - Type: `splitOut`
    - Connect input from `Merge1`.
    - Output to `Loop Over Items`.

11. **Create `Loop Over Items` node:**
    - Type: `splitInBatches`
    - Configure batch size as per performance needs.
    - Connect input from `Split Out`.
    - Create two outputs:
      - Output 1 to `Download Images`
      - Output 2 to `Image Gen (Nano Banana)`

12. **Create `Image Gen (Nano Banana)` node:**
    - Type: `httpRequest`
    - Configure with Nano Banana API endpoint.
    - Set authentication headers or tokens.
    - Connect input from `Loop Over Items` (output 2).
    - Output to `Wait1`.

13. **Create `Wait1` node:**
    - Type: `wait`
    - Configure wait duration suitable for Nano Banana processing time.
    - Connect input from `Image Gen (Nano Banana)`.
    - Output to `Get Image`.

14. **Create `Get Image` node:**
    - Type: `httpRequest`
    - Configure to poll or retrieve the processed image.
    - Connect input from `Wait1`.
    - Output to `Image created`.

15. **Create `Image created` node:**
    - Type: `if`
    - Configure condition to check if image generation was successful.
    - True output to `Loop Over Items` (to process next batch).
    - False output to `failed`.

16. **Create `failed` node:**
    - Type: `if`
    - Configure to decide retry or fallback strategy.
    - True output loops back to `Loop Over Items`.
    - False output to `Wait1` or workflow termination.

17. **Create `Download Images` node:**
    - Type: `httpRequest`
    - Configure to download final images.
    - Connect input from `Loop Over Items` (output 1).
    - Output to `Send Images`.

18. **Create `Send Images` node:**
    - Type: `telegram`
    - Configure Telegram bot credentials (OAuth2).
    - Set chat ID and message/media parameters.
    - Connect input from `Download Images`.
    - This is the final node.

19. **Create any sticky notes as reminders or documentation aids as needed.**

---

### 5. General Notes & Resources

| Note Content                                                                 | Context or Link                                    |
|------------------------------------------------------------------------------|---------------------------------------------------|
| The workflow integrates with Nano Banana API for image generation.           | Nano Banana API documentation (external reference) |
| Telegram node requires OAuth2 credentials linked to your bot account.        | https://core.telegram.org/bots/api                 |
| OpenAI node requires API key and appropriate model selection (GPT-4 recommended). | https://platform.openai.com/docs/models            |
| The form trigger webhook must be publicly accessible to receive submissions. | Ensure proper firewall and network configurations. |
| Wait node timing should be adjusted based on expected Nano Banana processing latency. | To optimize throughput and avoid unnecessary delays. |

---

**Disclaimer:** The provided text is extracted exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.