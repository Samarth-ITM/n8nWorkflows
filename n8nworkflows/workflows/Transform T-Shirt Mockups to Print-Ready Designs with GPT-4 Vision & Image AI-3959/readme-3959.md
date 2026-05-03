Transform T-Shirt Mockups to Print-Ready Designs with GPT-4 Vision & Image AI

https://n8nworkflows.xyz/workflows/transform-t-shirt-mockups-to-print-ready-designs-with-gpt-4-vision---image-ai-3959


# Transform T-Shirt Mockups to Print-Ready Designs with GPT-4 Vision & Image AI

### 1. Workflow Overview

This workflow automates the transformation of T-shirt mockup images into refined, print-ready designs using AI capabilities from OpenAI. It's designed for users who want to upgrade rough or outdated T-shirt design mockups into high-quality artworks optimized for printing on solid black backgrounds without any visible shirt or mockup framing.

The workflow is logically grouped into these blocks:

- **1.1 Input Reception and Validation:** Captures a user-submitted T-shirt mockup image URL and checks if it is valid.
- **1.2 Image Analysis with OpenAI Vision:** Sends the image URL to OpenAI’s GPT-4 vision model to extract design elements.
- **1.3 AI Prompt Refinement Agent:** Uses a custom AI agent to generate a refined, artistic prompt based on the image analysis.
- **1.4 Prompt Escaping for API Safety:** Cleans and escapes the refined prompt text to be safely sent to the image generation API.
- **1.5 Image Generation (gpt-image-1):** Calls OpenAI’s gpt-image-1 API to generate a new, print-ready T-shirt design.
- **1.6 Output Handling:** Converts the base64 image response into a binary file for downstream usage.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Validation

- **Overview:** This block listens for incoming chat messages containing image URLs and filters out messages that don't start with "https://", ensuring only valid image links proceed.
- **Nodes Involved:** `When chat message received`, `If`
- **Node Details:**

  - **When chat message received**
    - *Type:* Chat Trigger (n8n-nodes-langchain.chatTrigger)
    - *Role:* Entry point; waits for user chat input.
    - *Config:* Default webhook trigger; receives user messages (expected to contain image URLs).
    - *Connections:* Output → `If`
    - *Potential Failures:* Webhook not reachable; invalid or empty input.

  - **If**
    - *Type:* Conditional node (n8n-nodes-base.if)
    - *Role:* Checks if the chat input starts with "https://".
    - *Config:* Condition: `chatInput.startsWith("https://")`.
    - *Connections:* True → `OpenAI` (image analysis), False → no further action.
    - *Edge Cases:* Non-URL input, malformed URLs, non-image URLs.

---

#### 2.2 Image Analysis with OpenAI Vision

- **Overview:** Sends the validated image URL to OpenAI's GPT-4 vision API to analyze the design elements such as characters, text, layout, style, and tone.
- **Nodes Involved:** `OpenAI`
- **Node Details:**

  - **OpenAI**
    - *Type:* OpenAI Vision node (@n8n/n8n-nodes-langchain.openAi)
    - *Role:* Analyzes the input image to extract descriptive content.
    - *Config:*
      - Model: `gpt-4o` (GPT-4 vision model)
      - Operation: `analyze`
      - Image URL: The incoming chat input URL (hardcoded in node parameters here)
    - *Credentials:* OpenAI API key required.
    - *Connections:* Output → `AI Agent`
    - *Potential Failures:* API rate limit, invalid image URL, unsupported image format, API auth errors.

---

#### 2.3 AI Prompt Refinement Agent

- **Overview:** A specialized AI agent uses the analyzed image description to generate a concise, refined prompt that preserves key design elements but improves artistic quality, typography, and removes mockup elements.
- **Nodes Involved:** `AI Agent`
- **Node Details:**

  - **AI Agent**
    - *Type:* LangChain Agent node (@n8n/n8n-nodes-langchain.agent)
    - *Role:* Converts extracted image content into an enhanced, JSON-safe prompt.
    - *Config:*
      - Input Text: Uses `content` field from OpenAI vision output.
      - System Message: Detailed instructions for refinement emphasizing maintaining layout, enhancing typography, background set to solid black, output as a single escaped line.
      - Prompt Type: `define` (custom prompt generation)
    - *Connections:* Output → `Code` node for escaping.
    - *Potential Failures:* Unexpected input format, long text truncation, prompt generation errors.

---

#### 2.4 Prompt Escaping for API Safety

- **Overview:** This JavaScript function node cleans and escapes the refined prompt to prevent JSON formatting issues in the subsequent image generation call.
- **Nodes Involved:** `Code`
- **Node Details:**

  - **Code**
    - *Type:* Code node (n8n-nodes-base.code)
    - *Role:* Cleans line breaks, trims spaces, escapes backslashes and double quotes.
    - *Key Logic:* 
      - Replace `\n` with spaces.
      - Trim leading/trailing whitespace.
      - Escape backslashes (`\`) and double quotes (`"`).
      - Remove accidental leading/trailing escaped quotes.
    - *Connections:* Output → `HTTP Request`
    - *Potential Failures:* Unexpected input format, empty string output.

---

#### 2.5 Image Generation (gpt-image-1)

- **Overview:** Sends the escaped prompt to OpenAI’s `gpt-image-1` model to generate a new T-shirt artwork image optimized for printing.
- **Nodes Involved:** `HTTP Request`, `Split Out`, `Convert to File`
- **Node Details:**

  - **HTTP Request**
    - *Type:* HTTP Request node (n8n-nodes-base.httpRequest)
    - *Role:* Calls OpenAI API for image generation.
    - *Config:*
      - URL: `https://api.openai.com/v1/images/generations`
      - Method: POST
      - Body: JSON with model `gpt-image-1`, prompt from escaped string, size `1024x1536`, quality `high`.
      - Auth: Uses predefined OpenAI credentials.
    - *Connections:* Output → `Split Out`
    - *Edge Cases:* API rate limits, auth failures, invalid prompt errors, image generation failure.

  - **Split Out**
    - *Type:* Split Out node (n8n-nodes-base.splitOut)
    - *Role:* Extracts the base64-encoded image string from the API response.
    - *Config:* Field to split out: `data[0].b64_json`
    - *Connections:* Output → `Convert to File`
    - *Potential Failures:* Missing or malformed response data.

  - **Convert to File**
    - *Type:* Convert To File node (n8n-nodes-base.convertToFile)
    - *Role:* Converts base64 string to binary file for downstream use.
    - *Config:* Operation: `toBinary`, Source property: `data[0].b64_json`
    - *Connections:* Final output node.
    - *Potential Failures:* Conversion errors if base64 is corrupted.

---

#### 2.6 Optional Nodes (Chat Model & Sticky Notes)

- **OpenAI Chat Model**
  - Present but disconnected from main flow; may be used for alternative or future chat interactions.
- **Sticky Notes**
  - Provide explanatory comments for user guidance:
    - "Send a mockup image url to chat"
    - "Analyze image and generate new prompt"
    - "Generate the new Tshirt design"

---

### 3. Summary Table

| Node Name                 | Node Type                          | Functional Role                        | Input Node(s)              | Output Node(s)              | Sticky Note                                  |
|---------------------------|----------------------------------|-------------------------------------|----------------------------|-----------------------------|----------------------------------------------|
| When chat message received| Chat Trigger (@n8n/n8n-nodes-langchain.chatTrigger) | Entry point for T-shirt mockup URL input | -                          | If                          | Send a mockup image url to chat               |
| If                        | Conditional (n8n-nodes-base.if)  | Validates URL format                  | When chat message received | OpenAI                      |                                              |
| OpenAI                    | OpenAI Vision (@n8n/n8n-nodes-langchain.openAi) | Analyzes design elements from image | If                         | AI Agent                    | Analyze image and generate new prompt        |
| AI Agent                  | LangChain Agent (@n8n/n8n-nodes-langchain.agent) | Refines design prompt for generation | OpenAI                     | Code                        |                                              |
| Code                      | Code (n8n-nodes-base.code)        | Escapes prompt text for JSON safety  | AI Agent                   | HTTP Request                |                                              |
| HTTP Request              | HTTP Request (n8n-nodes-base.httpRequest) | Sends prompt to image generation API | Code                       | Split Out                   | Generate the new Tshirt design                |
| Split Out                 | Split Out (n8n-nodes-base.splitOut) | Extracts base64 image from API response | HTTP Request              | Convert to File             |                                              |
| Convert to File           | Convert To File (n8n-nodes-base.convertToFile) | Converts base64 string to binary file | Split Out                  | -                           |                                              |
| OpenAI Chat Model         | Chat Model (@n8n/n8n-nodes-langchain.lmChatOpenAi) | (Unused) Possibly alternative chat AI | -                          | AI Agent                    |                                              |
| Sticky Note               | Sticky Note (n8n-nodes-base.stickyNote) | User guidance comment                | -                          | -                           | Send a mockup image url to chat               |
| Sticky Note1              | Sticky Note (n8n-nodes-base.stickyNote) | User guidance comment                | -                          | -                           | Analyze image and generate new prompt         |
| Sticky Note2              | Sticky Note (n8n-nodes-base.stickyNote) | User guidance comment                | -                          | -                           | Generate the new Tshirt design                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node**
   - Add `Chat Trigger` node named `When chat message received`.
   - Configure webhook with default settings to receive chat messages.
   
2. **Add Conditional Node**
   - Add `If` node named `If`.
   - Connect `When chat message received` → `If`.
   - Set condition to check if incoming message (`chatInput`) starts with `https://`.

3. **Add OpenAI Vision Node**
   - Add LangChain `OpenAI` node.
   - Connect `If` (true output) → `OpenAI`.
   - Set model to `gpt-4o`.
   - Operation: `analyze`.
   - Configure to use incoming message URL as image URL input.
   - Add OpenAI API credentials (vision-enabled).

4. **Add AI Agent Node**
   - Add LangChain `AI Agent` node named `AI Agent`.
   - Connect `OpenAI` → `AI Agent`.
   - Configure prompt with system message instructing to refine T-shirt design prompt:
     - Preserve layout & key elements.
     - Enhance typography and artistic style.
     - Output escaped, single-line prompt suitable for JSON & image generation.
   - Pass analyzed content as input text.

5. **Add Code Node for Escaping**
   - Add Code node named `Code`.
   - Connect `AI Agent` → `Code`.
   - Insert JavaScript code to:
     - Replace line breaks with spaces.
     - Trim spaces.
     - Escape backslashes and quotes.
     - Remove accidental leading/trailing escaped quotes.
   - Output escaped prompt as `escapedString`.

6. **Add HTTP Request Node for Image Generation**
   - Add `HTTP Request` node named `HTTP Request`.
   - Connect `Code` → `HTTP Request`.
   - Configure:
     - URL: `https://api.openai.com/v1/images/generations`
     - Method: POST
     - Body (JSON):
       ```json
       {
         "model": "gpt-image-1",
         "prompt": "{{ $json.escapedString }}",
         "n": 1,
         "size": "1024x1536",
         "quality": "high"
       }
       ```
     - Authentication: Use OpenAI API credentials with image generation rights.

7. **Add Split Out Node**
   - Add `Split Out` node named `Split Out`.
   - Connect `HTTP Request` → `Split Out`.
   - Set field to split out: `data[0].b64_json` (extract image base64).

8. **Add Convert To File Node**
   - Add `Convert To File` node named `Convert to File`.
   - Connect `Split Out` → `Convert to File`.
   - Operation: `toBinary`.
   - Source Property: `data[0].b64_json`.

9. **(Optional) Add Sticky Notes**
   - Add sticky notes near relevant nodes for documentation:
     - Next to chat trigger: "Send a mockup image url to chat"
     - Next to OpenAI vision and AI Agent nodes: "Analyze image and generate new prompt"
     - Next to HTTP Request: "Generate the new Tshirt design"

10. **Configure Credentials**
    - Set up OpenAI API credentials twice if needed:
      - One for GPT-4 vision (`gpt-4o`).
      - Another for image generation (`gpt-image-1`).

11. **Test Workflow**
    - Deploy and trigger workflow by sending a chat message with a valid T-shirt mockup image URL.
    - Verify refined prompt generation and resulting image output.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                      | Context or Link                                                |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------|
| This workflow uses OpenAI's paid `gpt-image-1` model to generate images, with an estimated cost of ~$0.25 per image at 1024x1536 resolution.                     | https://platform.openai.com/docs/models/gpt-image-1           |
| Ensure API keys have permission for both vision and image generation endpoints.                                                                                  | OpenAI API dashboard                                           |
| The AI Agent’s system message is carefully crafted to maintain original design elements and produce a JSON-safe prompt suitable for automated image generation. | Internal prompt design best practices                          |
| The workflow is modular and can be extended by adding nodes to send results to Telegram, Notion, or other systems after image generation.                       | n8n automation & integration best practices                    |
| Use solid black backgrounds in final artwork to optimize print quality and avoid mockup elements like collars or shadows.                                        | Design guideline for print-ready T-shirt graphics             |

---

This document provides a complete, clear, and detailed reference to understand, reproduce, and maintain the "Transform T-Shirt Mockups to Print-Ready Designs with GPT-4 Vision & Image AI" workflow in n8n.