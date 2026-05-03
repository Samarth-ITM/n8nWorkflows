Generate Multilingual Shopify Product Descriptions with Gemini 2.5 Vision AI

https://n8nworkflows.xyz/workflows/generate-multilingual-shopify-product-descriptions-with-gemini-2-5-vision-ai-8179


# Generate Multilingual Shopify Product Descriptions with Gemini 2.5 Vision AI

### 1. Workflow Overview

This workflow automates the generation and multilingual translation of Shopify product descriptions using Google Gemini 2.5 Vision AI. It targets e-commerce use cases where product data and images are fetched from Shopify, analyzed by AI to extract objective visual descriptions, then combined with existing product details to generate polished, conversion-friendly product copy in six languages (Spanish, German, English, French, Italian, Portuguese). The workflow outputs these multilingual descriptions into a Google Sheet for further use.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Manual trigger to start the workflow.
- **1.2 Shopify Product Retrieval:** Fetch multiple products from Shopify (limited to 2 for testing).
- **1.3 Image Analysis with Vision AI:** Analyze the main product image using Google Gemini Vision to extract detailed, objective visual traits.
- **1.4 AI Product Description Generation:** Use a LangChain AI Agent with a Google Gemini Chat Model to generate multilingual product descriptions based on the Shopify product data and the Vision AI analysis.
- **1.5 Structured Output Parsing:** Parse AI output strictly into a JSON format with six languages and defined keys.
- **1.6 Data Expansion and Sanitization:** Transform parsed output into rows per language with additional metadata.
- **1.7 Output Storage:** Append the multilingual descriptions into a Google Sheet for recordkeeping or further processing.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** Initiates the workflow manually.
- **Nodes Involved:** 
  - When clicking ‘Execute workflow’
- **Node Details:**
  - **When clicking ‘Execute workflow’**
    - Type: Manual Trigger
    - Role: Starts the workflow on user command.
    - Configuration: Default manual trigger, no parameters.
    - Input/Output: No input; output triggers the next node.
    - Edge Cases: None significant; manual start prevents unintended runs.

#### 2.2 Shopify Product Retrieval

- **Overview:** Fetches product data from Shopify using API credentials.
- **Nodes Involved:**
  - Get many products
- **Node Details:**
  - **Get many products**
    - Type: Shopify node
    - Role: Retrieves up to 2 products from the Shopify store.
    - Configuration:
      - Operation: getAll products
      - Limit: 2 (for testing or batch processing)
      - Authentication: OAuth2 access token
    - Input: Trigger from manual start.
    - Output: JSON array of Shopify product objects, including fields like id, title, images, vendor, product_type, body_html, handle, variants, etc.
    - Failure Types:
      - Auth errors if token invalid or expired.
      - Rate limits from Shopify API.
      - Network timeouts.
    - Notes: Limited to 2 products for manageable processing.

#### 2.3 Image Analysis with Vision AI

- **Overview:** Uses Google Gemini Vision AI to analyze the first image of each product, extracting objective visual product descriptions.
- **Nodes Involved:**
  - Analyze image
- **Node Details:**
  - **Analyze image**
    - Type: LangChain Google Gemini Vision AI node
    - Role: Analyze product image URL, return a detailed, factual product description.
    - Configuration:
      - Text prompt instructs to focus on observable product traits only.
      - Output: Short paragraph (3-5 sentences) describing product type, materials, colors, patterns, distinctive elements.
      - Image URL: Pulled dynamically from the first product image source.
      - Model: "models/gemini-2.5-flash-lite"
      - Credentials: Google Palm API credentials.
    - Input: Product data from Shopify node.
    - Output: Text description of product image.
    - Failure Types:
      - Invalid or missing image URL.
      - API authentication or quota errors.
      - Image too complex or ambiguous, yielding poor descriptions.
    - Notes:
      - Explicit instructions prevent hallucinations or assumptions.

#### 2.4 AI Product Description Generation

- **Overview:** Combines Shopify product data and vision analysis to generate carefully crafted, multilingual product descriptions using a LangChain AI Agent with Google Gemini Chat.
- **Nodes Involved:**
  - AI Agent
  - Google Gemini Chat Model1
  - Structured Output Parser
- **Node Details:**
  - **AI Agent**
    - Type: LangChain Agent node
    - Role: Constructs multilingual product descriptions using inputs and system prompt.
    - Configuration:
      - Dynamic prompt with embedded product data and vision description.
      - System message enforces strict output formatting, multilingual terminology, anti-hallucination rules, HTML formatting constraints, and output JSON schema.
      - Output: JSON object with keys for six languages, each containing product_name, description, and handle.
      - Input: Receives text from Analyze image node and Shopify product data.
      - Output: Raw AI output JSON string.
      - Failure Types:
        - Parsing or formatting errors in prompt or output.
        - API errors or timeouts.
        - Model hallucinations if prompt misunderstood.
  - **Google Gemini Chat Model1**
    - Type: LangChain Google Gemini Chat Language Model node
    - Role: Provides the underlying language model with configured temperature and token limits.
    - Configuration:
      - Temperature: 0.4 for balanced creativity.
      - Max output tokens: 4048.
      - Credentials: Google Palm API.
    - Input/Output: Connected as language model for AI Agent.
  - **Structured Output Parser**
    - Type: LangChain Output Parser node
    - Role: Parses AI Agent’s raw JSON response strictly into structured JSON.
    - Configuration:
      - JSON schema example ensures six languages with required keys.
    - Input: AI Agent output.
    - Output: Parsed JSON object.
    - Failure Types:
      - Parsing failures if AI response deviates from strict schema.
      - Missing keys or malformed JSON.

#### 2.5 Data Expansion and Sanitization

- **Overview:** Converts parsed multilingual JSON into separate items per language, attaching product IDs and base handles for database or sheet insertion.
- **Nodes Involved:**
  - Expand Languages & Sanitize
- **Node Details:**
  - **Expand Languages & Sanitize**
    - Type: Code Node (JavaScript)
    - Role: Converts the JSON object with multiple languages into an array of objects for each language with product metadata.
    - Configuration:
      - Iterates over languages [es, de, en, fr, it, pt].
      - Builds new JSON items with product_id, lang, handle, shopify_product_name, shopify_description, base_handle_es.
      - Uses product id and base handle from Shopify node.
    - Input: Parsed JSON from Structured Output Parser.
    - Output: Array of objects, each representing one language’s product description.
    - Failure Types:
      - Missing or undefined keys in input JSON.
      - JavaScript errors if input structure changes.
    - Notes:
      - Provides uniform data shape for Google Sheets node.

#### 2.6 Output Storage

- **Overview:** Appends the multilingual product descriptions into a specified Google Sheet.
- **Nodes Involved:**
  - Append row in sheet
- **Node Details:**
  - **Append row in sheet**
    - Type: Google Sheets node
    - Role: Appends rows with multilingual product descriptions and metadata into a Google Sheet.
    - Configuration:
      - Document ID and sheet name (gid=0) set to specific spreadsheet.
      - Columns auto-mapped from input JSON keys: product_id, lang, handle, shopify_product_name, shopify_description, base_handle_es.
      - Authentication: Service Account for Google APIs.
    - Input: Expanded and sanitized language items.
    - Output: Confirmation of appended rows.
    - Failure Types:
      - Authentication failures if service account invalid.
      - API quota or rate limits.
      - Schema mismatch if columns do not exist or types incompatible.
    - Notes:
      - Allows external review or further processing of generated descriptions.

---

### 3. Summary Table

| Node Name                  | Node Type                                  | Functional Role                              | Input Node(s)              | Output Node(s)             | Sticky Note                                               |
|----------------------------|--------------------------------------------|----------------------------------------------|----------------------------|----------------------------|-----------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger                             | Starts workflow manually                      | —                          | Get many products          |                                                           |
| Get many products           | Shopify                                    | Retrieves products from Shopify API          | When clicking ‘Execute workflow’ | Analyze image             |                                                           |
| Analyze image              | LangChain Google Gemini Vision AI          | Analyzes product image to extract visual traits | Get many products           | AI Agent                   |                                                           |
| AI Agent                  | LangChain Agent                             | Generates multilingual product descriptions  | Analyze image, Google Gemini Chat Model1, Structured Output Parser (ai_outputParser) | Expand Languages & Sanitize |                                                           |
| Google Gemini Chat Model1  | LangChain Google Gemini Chat Model          | Language model for AI Agent                   | —                          | AI Agent                   |                                                           |
| Structured Output Parser   | LangChain Output Parser                      | Parses AI JSON output into structured JSON   | AI Agent                   | AI Agent                   |                                                           |
| Expand Languages & Sanitize | Code Node                                  | Expands multilingual JSON to individual language items | AI Agent                   | Append row in sheet        |                                                           |
| Append row in sheet        | Google Sheets                              | Appends multilingual descriptions into Google Sheet | Expand Languages & Sanitize | —                          |                                                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create manual trigger node:**
   - Add node: Manual Trigger
   - Name: "When clicking ‘Execute workflow’"
   - No parameters needed.

2. **Add Shopify node to get products:**
   - Add node: Shopify
   - Name: "Get many products"
   - Operation: Get All Products
   - Limit: 2 (for testing)
   - Authentication: Configure Shopify Access Token OAuth2 credentials.
   - Connect output of manual trigger to this node.

3. **Add Vision AI node to analyze product images:**
   - Add node: LangChain Google Gemini Vision AI (type: @n8n/n8n-nodes-langchain.googleGemini)
   - Name: "Analyze image"
   - Operation: Analyze Image
   - Model: "models/gemini-2.5-flash-lite"
   - Text prompt: Provide detailed instructions to analyze visible product traits only, per the original prompt.
   - Image URLs: Set to expression `{{$json.images[0].src}}` from Shopify product data.
   - Credentials: Configure Google Palm API credentials.
   - Connect output of “Get many products” to this node.

4. **Add AI Agent node for multilingual copy generation:**
   - Add node: LangChain Agent node (type: @n8n/n8n-nodes-langchain.agent)
   - Name: "AI Agent"
   - Text input: Construct dynamic prompt embedding:
     - Product image URL
     - Brand (vendor)
     - Product name (title)
     - User draft description (body_html)
     - Product type
     - Vision description (from previous node)
   - System message: Use the detailed multilingual professional copywriting prompt with strict JSON output format, as in the original.
   - Credentials: Google Palm API
   - Connect output of “Analyze image” to this node.

5. **Add Google Gemini Chat Model node:**
   - Add node: LangChain Google Gemini Chat Model (type: @n8n/n8n-nodes-langchain.lmChatGoogleGemini)
   - Name: "Google Gemini Chat Model1"
   - Parameters:
     - Temperature: 0.4
     - Max output tokens: 4048
   - Credentials: Google Palm API
   - Connect this node as language model for "AI Agent" node.

6. **Add Structured Output Parser node:**
   - Add node: LangChain Output Parser (type: @n8n/n8n-nodes-langchain.outputParserStructured)
   - Name: "Structured Output Parser"
   - JSON Schema Example: Define keys for es, de, en, fr, it, pt each with shopify_product_name, shopify_description, handle.
   - Connect ai_outputParser output of "AI Agent" node to this node.

7. **Add Code node to expand and sanitize languages:**
   - Add node: Code (JavaScript)
   - Name: "Expand Languages & Sanitize"
   - JavaScript code: Iterate over each language key, build new JSON objects including product_id, language, handle, shopify product name, description, and base Spanish handle.
   - Connect output of "AI Agent" node (main) to this node.

8. **Add Google Sheets node to append rows:**
   - Add node: Google Sheets
   - Name: "Append row in sheet"
   - Operation: Append
   - Document ID: Set to target Google Sheet ID
   - Sheet Name: Set to target sheet (gid=0)
   - Columns: Auto-map fields from input JSON (product_id, lang, handle, shopify_product_name, shopify_description, base_handle_es)
   - Authentication: Configure Google Sheets Service Account credentials.
   - Connect output of "Expand Languages & Sanitize" node to this node.

9. **Connect nodes in order:**
   - Manual Trigger → Get many products → Analyze image → AI Agent → Expand Languages & Sanitize → Append row in sheet
   - Google Gemini Chat Model → AI Agent (language model input)
   - Structured Output Parser → AI Agent (ai_outputParser)

10. **Test the workflow:**
    - Execute manually.
    - Verify outputs in Google Sheet.
    - Handle errors such as missing images, API auth failures, AI response parsing issues.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               | Context or Link                                                                                                  |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| The AI prompt enforces strict multilingual terminology for footwear across ES, DE, EN, FR, IT, PT to maintain industry standard language and avoid hallucinations.                                                                                                                                                                                                                                                                                                                                                                         | Embedded system message in AI Agent node.                                                                       |
| Output format is strictly JSON with no extraneous text to facilitate automation and integration.                                                                                                                                                                                                                                                                                                                                                                                                                                         | System prompt instructions.                                                                                      |
| Google Sheets are updated using a Service Account credential to enable automated, secure access without manual login.                                                                                                                                                                                                                                                                                                                                                                                                                    | Google Sheets node configuration.                                                                                |
| The workflow emphasizes objective image analysis by Gemini Vision AI, preventing assumptions about brand, size, or comfort.                                                                                                                                                                                                                                                                                                                                                                                                              | Analyze image node prompt.                                                                                       |
| Shopify API rate limits and token expiration should be monitored to avoid interruptions.                                                                                                                                                                                                                                                                                                                                                                                                                                                  | Shopify node considerations.                                                                                      |
| For more on Google Gemini and PaLM API integration in n8n, consult https://docs.n8n.io/integrations/builtin/n8n-nodes-langchain/google-gemini/                                                                                                                                                                                                                                                                                                                                                                                          | Official n8n LangChain Google Gemini documentation.                                                              |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects content policies and contains no illegal, offensive, or protected elements. All processed data is legal and public.