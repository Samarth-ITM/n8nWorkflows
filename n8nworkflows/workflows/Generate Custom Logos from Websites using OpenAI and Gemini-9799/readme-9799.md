Generate Custom Logos from Websites using OpenAI and Gemini

https://n8nworkflows.xyz/workflows/generate-custom-logos-from-websites-using-openai-and-gemini-9799


# Generate Custom Logos from Websites using OpenAI and Gemini

### 1. Workflow Overview

This workflow automates the generation of custom logos from a given website URL by leveraging AI technologies. Upon receiving a website URL via webhook, it performs two parallel data captures: a visual screenshot and the website’s HTML content. These inputs feed into an AI language model that crafts a detailed prompt for logo creation. The prompt is then sent to Google Gemini AI to generate a logo image, which is finally returned as a binary response.

**Target Use Cases:**

- Marketing and branding teams needing quick logo prototypes from client websites  
- Developers or designers automating logo generation for websites  
- Educational demos showing AI-driven multimodal design workflows  

**Logical Blocks:**

- **1.1 Input Reception:** Receives website URL via Webhook  
- **1.2 Data Acquisition:** Captures website screenshot and fetches HTML content  
- **1.3 AI Prompt Generation:** Uses OpenAI GPT-5 mini to generate logo prompt from inputs  
- **1.4 Logo Image Generation:** Uses Google Gemini to create logo image from prompt  
- **1.5 Response Delivery:** Returns generated logo image to the requester  

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  Listens for incoming POST requests containing a website URL. Triggers the workflow and passes the URL downstream.

- **Nodes Involved:**  
  - When Website URL Received

- **Node Details:**  

  **When Website URL Received**  
  - Type: Webhook Trigger  
  - Configuration:  
    - HTTP Method: POST  
    - Path: `175b3350-1d0c-48c5-99d1-06e3b209a68097t657yi97` (custom webhook endpoint)  
    - Response Mode: Response node (waits for downstream respondToWebhook node)  
  - Key Expressions/Variables: Reads `websiteUrl` from JSON body (`$json.body.websiteUrl`)  
  - Connections: Output to "Capture Website Screenshot"  
  - Edge Cases:  
    - Missing or malformed `websiteUrl` in payload  
    - Unauthorized access (depends on webhook visibility and security settings)  
  - Notes: Expects JSON body format: `{"websiteUrl": "https://example.com"}`

---

#### 1.2 Data Acquisition

- **Overview:**  
  Acquires two types of data from the website URL: a screenshot for visual context and the raw HTML content for textual analysis.

- **Nodes Involved:**  
  - Capture Website Screenshot  
  - Fetch Website Content

- **Node Details:**  

  **Capture Website Screenshot**  
  - Type: HTTP Request  
  - Configuration:  
    - URL: ScreenshotOne API endpoint (`https://api.screenshotone.com/take`)  
    - Method: GET (default for HTTP Request)  
    - Query Parameters include:  
      - `access_key`: Placeholder to be replaced with user’s ScreenshotOne API key  
      - `url`: Dynamic, from input webhook (`{{$json.body.websiteUrl}}`)  
      - Format: JPG  
      - Various ad/cookie/banner blocking enabled  
      - Timeout: 60 seconds  
      - Image quality: 80  
    - Output: JSON containing `screenshot_url`  
  - Connections: Output to "Fetch Website Content"  
  - Edge Cases:  
    - API key missing or invalid (auth failure)  
    - Website inaccessible or slow (timeout)  
    - Screenshot API rate limits  
  - Notes: Replace placeholder with valid ScreenshotOne access key  

  **Fetch Website Content**  
  - Type: HTTP Request  
  - Configuration:  
    - URL: Dynamic, from webhook input (`{{$json.body.websiteUrl}}`)  
    - Method: GET  
    - No special headers or auth configured  
    - Output: Raw HTML content of the website page in response body  
  - Connections: Output to "Generate Logo Prompt"  
  - Edge Cases:  
    - Website rejects request (e.g., bot protection)  
    - Empty or malformed HTML response  
    - HTTP errors (404, 500, etc.)  

---

#### 1.3 AI Prompt Generation

- **Overview:**  
  Uses OpenAI GPT-5 mini language model to generate a detailed prompt for logo creation based on the website’s screenshot URL and HTML content.

- **Nodes Involved:**  
  - GPT-5 mini (Language Model)  
  - Generate Logo Prompt (LangChain Agent Node)

- **Node Details:**  

  **GPT-5 mini**  
  - Type: AI Language Model (OpenAI)  
  - Configuration:  
    - Model: `gpt-5-mini` chosen from available OpenAI models  
    - No additional options set  
  - Credentials: Requires valid OpenAI API key  
  - Connections: Output to "Generate Logo Prompt" node  
  - Edge Cases:  
    - API key invalid or quota exceeded  
    - Model unavailable or rate limited  

  **Generate Logo Prompt**  
  - Type: LangChain Agent Node (OpenAI integration)  
  - Configuration:  
    - Prompt text contains instructions to create a logo prompt using screenshot URL and website HTML content  
    - Injected variables:  
      - Screenshot URL from previous screenshot node (`$('Capture Website Screenshot').item.json.screenshot_url`)  
      - Website page content (`$json.data` from Fetch Website Content)  
    - Options: Enables passthrough of binary images (for potential multimodal processing)  
  - Connections: Output to "Generate Logo Image"  
  - Edge Cases:  
    - Empty or invalid inputs leading to poor prompt generation  
    - Expression evaluation errors if referenced nodes fail or have no data  

---

#### 1.4 Logo Image Generation

- **Overview:**  
  Takes the AI-generated prompt and sends it to Google Gemini AI to generate a logo image, returning binary image data.

- **Nodes Involved:**  
  - Generate Logo Image

- **Node Details:**  

  **Generate Logo Image**  
  - Type: LangChain Google Gemini Node (Image generation)  
  - Configuration:  
    - Prompt: From previous node output (`$json.output`)  
    - Model: `models/gemini-2.5-flash-image` (Google Gemini image generation model)  
    - Resource type: Image  
  - Credentials: Google PaLM API credentials required  
  - Connections: Output to "Respond with Logo"  
  - Edge Cases:  
    - Invalid or empty prompt leads to no or poor images  
    - API quota or authentication errors  
    - Latency or timeouts during image generation  

---

#### 1.5 Response Delivery

- **Overview:**  
  Sends the generated logo image as a binary response to the original webhook caller.

- **Nodes Involved:**  
  - Respond with Logo

- **Node Details:**  

  **Respond with Logo**  
  - Type: Respond to Webhook  
  - Configuration:  
    - Response Mode: Binary response (image file)  
  - Connections: None (terminal node)  
  - Edge Cases:  
    - If input binary data is missing or corrupted, response will fail  
    - Client timeouts or connectivity issues  

---

### 3. Summary Table

| Node Name                 | Node Type                              | Functional Role                       | Input Node(s)              | Output Node(s)           | Sticky Note                                                                                                           |
|---------------------------|--------------------------------------|-------------------------------------|----------------------------|--------------------------|-----------------------------------------------------------------------------------------------------------------------|
| When Website URL Received  | Webhook                              | Receives website URL input           |                            | Capture Website Screenshot| ## 📥 When Website URL Received<br>**Purpose:** Triggers workflow on POST request with website URL.<br>**Note:** Body format: {"websiteUrl": "https://example.com"} |
| Capture Website Screenshot | HTTP Request                         | Captures site screenshot via API    | When Website URL Received   | Fetch Website Content      | ## 🖼️ Capture Website Screenshot<br>**Purpose:** Fetches site screenshot via ScreenshotOne API for visual analysis.<br>**Note:** Replace placeholder with your API key; outputs JSON with screenshot_url. |
| Fetch Website Content      | HTTP Request                         | Fetches HTML content from site      | Capture Website Screenshot  | Generate Logo Prompt       | ## 🌐 Fetch Website Content<br>**Purpose:** Scrapes HTML from the URL for text-based site analysis.                    |
| GPT-5 mini                | AI Language Model (OpenAI)           | Provides AI model for prompt gen    |                            | Generate Logo Prompt       |                                                                                                                       |
| Generate Logo Prompt       | LangChain Agent Node (OpenAI)        | Generates logo prompt from inputs   | Fetch Website Content, GPT-5 mini | Generate Logo Image        | ## ✍️ Generate Logo Prompt<br>**Purpose:** AI agent crafts logo prompt using OpenAI from content and screenshot.<br>**Note:** Multimodal input; outputs refined prompt for image gen. |
| Generate Logo Image        | LangChain Google Gemini Node (Image) | Generates logo image from prompt    | Generate Logo Prompt        | Respond with Logo          | ## 🎨 Generate Logo Image<br>**Purpose:** Creates logo via Google Gemini using the AI-crafted prompt.<br>**Note:** Image resource; returns binary data for response. |
| Respond with Logo          | Respond to Webhook                   | Sends back generated logo image     | Generate Logo Image         |                          |                                                                                                                       |
| Note: Webhook Trigger      | Sticky Note                         | Documentation                       |                            |                          | ## 📥 When Website URL Received<br>**Purpose:** Triggers workflow on POST request with website URL.<br>**Note:** Body format: {"websiteUrl": "https://example.com"} |
| Note: Screenshot Capture   | Sticky Note                         | Documentation                       |                            |                          | ## 🖼️ Capture Website Screenshot<br>**Purpose:** Fetches site screenshot via ScreenshotOne API for visual analysis.<br>**Note:** Replace placeholder with your API key; outputs JSON with screenshot_url. |
| Note: Content Fetch        | Sticky Note                         | Documentation                       |                            |                          | ## 🌐 Fetch Website Content<br>**Purpose:** Scrapes HTML from the URL for text-based site analysis.                    |
| Note: Prompt Generation    | Sticky Note                         | Documentation                       |                            |                          | ## ✍️ Generate Logo Prompt<br>**Purpose:** AI agent crafts logo prompt using OpenAI from content and screenshot.<br>**Note:** Multimodal input; outputs refined prompt for image gen. |
| Note: Logo Generation      | Sticky Note                         | Documentation                       |                            |                          | ## 🎨 Generate Logo Image<br>**Purpose:** Creates logo via Google Gemini using the AI-crafted prompt.<br>**Note:** Image resource; returns binary data for response. |
| Overview Note5             | Sticky Note                         | Documentation Overview              |                            |                          | # 🤖 AI Logo Generator from Website URL<br>## 📋 What This Template Does<br>This workflow receives a website URL via webhook, captures a screenshot and fetches page content, uses OpenAI to craft a logo prompt from visuals and text, then generates the image with Google Gemini for binary response.<br>## 🔧 Prerequisites<br>- n8n instance with webhook support<br>- ScreenshotOne account<br>- OpenAI account<br>- Google AI Studio account<br>## 🔑 Required Credentials<br>### ScreenshotOne API Setup<br>1. Sign up at screenshotone.com → Dashboard → API Keys<br>2. Generate access key<br>3. Replace placeholder in "Capture Website Screenshot" node<br>### OpenAI API Setup<br>1. platform.openai.com → API Keys<br>2. Create secret key<br>3. Add as "OpenAI API" credential<br>### Google Gemini API Setup<br>1. aistudio.google.com/app/apikey<br>2. Create API key<br>3. Add as "Google PaLM API" credential<br>## ⚙️ Configuration Steps<br>1. Import JSON workflow<br>2. Assign credentials to nodes<br>3. Replace API key placeholder<br>4. Activate webhook<br>5. Test with POST {"websiteUrl": "https://example.com"}<br>## 🎯 Use Cases<br>- Marketing: Generate client logo prototypes<br>- Developers: Auto-match logos for sites<br>- Designers: Inspire from competitors<br>- Education: Demo AI design<br>## ⚠️ Troubleshooting<br>- Screenshot timeout: Increase to 120s, check URL<br>- Empty prompt: Verify OpenAI quota<br>- Blank logo: Add style to prompt, check limits<br>- No trigger: Confirm POST JSON body |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Add a **Webhook** node named `When Website URL Received`.  
   - Set HTTP Method to `POST`.  
   - Define a unique path, e.g., `175b3350-1d0c-48c5-99d1-06e3b209a68097t657yi97`.  
   - Set Response Mode to `Response Node` (to wait for downstream response).  
   - This node will receive JSON payload with `websiteUrl` field.

2. **Create HTTP Request Node for Screenshot**  
   - Add an **HTTP Request** node named `Capture Website Screenshot`.  
   - Set Method to GET.  
   - URL: `https://api.screenshotone.com/take`.  
   - Add Query Parameters:  
     - `access_key`: Your ScreenshotOne API key (replace placeholder).  
     - `url`: Expression referencing webhook input URL: `{{$json.body.websiteUrl}}`.  
     - `format`: `jpg`  
     - `block_ads`: `true`  
     - `block_cookie_banners`: `true`  
     - `block_banners_by_heuristics`: `false`  
     - `block_trackers`: `true`  
     - `delay`: `0`  
     - `timeout`: `60` (seconds)  
     - `response_type`: `json`  
     - `image_quality`: `80`  
   - Connect output of `When Website URL Received` to this node.

3. **Create HTTP Request Node for Website Content**  
   - Add an **HTTP Request** node named `Fetch Website Content`.  
   - Set Method to GET.  
   - URL: Expression: `{{$node["When Website URL Received"].json.body.websiteUrl}}`.  
   - No authentication needed.  
   - Connect output of `Capture Website Screenshot` to this node.

4. **Create AI Language Model Node (OpenAI GPT-5 mini)**  
   - Add a **Language Model (OpenAI)** node named `GPT-5 mini`.  
   - Select model: `gpt-5-mini`.  
   - Configure OpenAI API credentials (add your OpenAI API key as credential).  
   - No extra options needed.  
   - No direct input connection (connected in next step).

5. **Create LangChain Agent Node for Prompt Generation**  
   - Add a **LangChain Agent** node named `Generate Logo Prompt`.  
   - Use prompt type: `Define`.  
   - Set prompt text as:  
     ```
     Write a prompt to create a logo for this website based of the content of the site and a screenshot of the front page

     Screenshot: {{ $('Capture Website Screenshot').item.json.screenshot_url }}
     Website data:{{ $json.data }}

     Output the prompt in "Prompt"
     ```  
   - Enable passthrough binary images option.  
   - Connect input from both `Fetch Website Content` and `GPT-5 mini` nodes.  
   - Configure credentials if needed for OpenAI (should be linked to `GPT-5 mini`).

6. **Create Google Gemini Node for Logo Image Generation**  
   - Add a **LangChain Google Gemini** node named `Generate Logo Image`.  
   - Set prompt input to the output from `Generate Logo Prompt` node (`{{$json.output}}`).  
   - Select model ID: `models/gemini-2.5-flash-image`.  
   - Set resource type to `image`.  
   - Configure Google PaLM API credentials with your Google AI Studio API key.  
   - Connect output from `Generate Logo Prompt` to this node.

7. **Create Respond to Webhook Node**  
   - Add a **Respond to Webhook** node named `Respond with Logo`.  
   - Set response type to `binary` to send back the generated image data.  
   - Connect output from `Generate Logo Image` to this node.

8. **Connect Node Flow**  
   - `When Website URL Received` → `Capture Website Screenshot` → `Fetch Website Content` → `Generate Logo Prompt` → `Generate Logo Image` → `Respond with Logo`  
   - Also connect `GPT-5 mini` to `Generate Logo Prompt` for language model input.

9. **Final Setup**  
   - Replace ScreenshotOne API key placeholder with your valid key.  
   - Add and assign OpenAI API credentials to `GPT-5 mini` node.  
   - Add and assign Google PaLM API credentials to `Generate Logo Image` node.  
   - Activate the webhook node and test by sending a POST request with JSON body:  
     ```json
     { "websiteUrl": "https://example.com" }
     ```  
   - Validate that the response contains a binary image of the generated logo.

---

### 5. General Notes & Resources

| Note Content                                                                                                                        | Context or Link                              |
|------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------|
| This workflow requires accounts and API keys for ScreenshotOne, OpenAI, and Google AI Studio (PaLM API). Ensure to secure these keys. | Credential setup instructions in Overview Note5 node. |
| Troubleshooting tips include increasing screenshot timeout to 120s, verifying API quotas, adding style descriptors to logo prompt, and confirming webhook payload format. | Found in Overview Note5 node content. |
| Use this workflow for marketing prototyping, developer automation, design inspiration, and educational AI demos.                  | Use case section of Overview Note5 node.    |
| ScreenshotOne API documentation: https://screenshotone.com/docs                                                                    | ScreenshotOne integration reference          |
| OpenAI API docs: https://platform.openai.com/docs                                                                                   | AI prompt generation reference                |
| Google PaLM (Gemini) API docs: https://developers.generativeai.google/api/python/latest/                                            | Image generation reference                     |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This process strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All handled data are legal and public.