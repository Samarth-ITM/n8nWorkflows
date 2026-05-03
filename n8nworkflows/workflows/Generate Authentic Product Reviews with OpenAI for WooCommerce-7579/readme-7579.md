Generate Authentic Product Reviews with OpenAI for WooCommerce

https://n8nworkflows.xyz/workflows/generate-authentic-product-reviews-with-openai-for-woocommerce-7579


# Generate Authentic Product Reviews with OpenAI for WooCommerce

### 1. Workflow Overview

This workflow automates the generation and posting of authentic-feeling product reviews for a WooCommerce store using OpenAI's language model. It is designed for e-commerce managers or marketers who want to enrich their product pages with diverse, natural-sounding customer comments without manual input. The workflow is logically divided into these main blocks:

- **1.1 Manual Trigger:** Starts the workflow execution on demand.
- **1.2 WooCommerce Product Retrieval:** Fetches a batch of products from the WooCommerce REST API.
- **1.3 AI Prompt Construction:** Builds a customized AI prompt for each product, incorporating product details.
- **1.4 AI Review Generation:** Sends the prompt to OpenAI to generate a short, conversational product review.
- **1.5 AI Output Extraction:** Extracts the generated review text from the AI response.
- **1.6 Posting Review to WooCommerce:** Posts the generated review back to the WooCommerce store as a product comment with randomized reviewer details.

---

### 2. Block-by-Block Analysis

#### 1.1 Manual Trigger

- **Overview:**  
  Initiates the workflow manually when a user clicks ‘Execute workflow’ in the n8n UI.

- **Nodes Involved:**  
  - `When clicking ‘Execute workflow’`

- **Node Details:**  
  - Type: Manual Trigger  
  - Configuration: Default, no parameters required.  
  - Inputs: None  
  - Outputs: Connects to “Fetch Products from WooCommerce” node.  
  - Edge Cases: None typical; workflow will not run unless manually triggered.  
  - Version-specific: n8n standard manual trigger node.  

#### 1.2 WooCommerce Product Retrieval

- **Overview:**  
  Fetches up to 100 products from the WooCommerce REST API, using HTTP Basic Authentication. The products include essential fields like name and descriptions, which will be used to generate AI prompts.

- **Nodes Involved:**  
  - `Fetch Products from WooCommerce`

- **Node Details:**  
  - Type: HTTP Request  
  - Configuration:  
    - URL: `https://example.com/wp-json/wc/v3/products?per_page=100` (replace `example.com` with your store domain)  
    - Method: GET  
    - Authentication: HTTP Basic Auth with API keys (credentials stored in n8n)  
    - Timeout: 600,000 ms (10 minutes)  
    - Batching: batch size 100 (fetches in batches if more than 100 products exist)  
    - Retry: up to 5 times with 2000 ms delay between attempts  
  - Inputs: Triggered from manual trigger.  
  - Outputs: Sends all product items to the next block for prompt generation.  
  - Edge Cases:  
    - Authentication failures (invalid API keys)  
    - Network timeouts or server errors  
    - Pagination: Currently, only one batch of 100 products fetched; if more are needed, workflow extension required.  
  - Version-specific: Uses n8n HTTP Request node v4.2 with batching enabled.  

#### 1.3 AI Prompt Construction

- **Overview:**  
  Constructs a tailored AI prompt string for each product to guide the AI in generating a natural, varied product review.

- **Nodes Involved:**  
  - `Build Product Comment Prompt`

- **Node Details:**  
  - Type: Set  
  - Configuration:  
    - Creates a field named `prompt` with a template string combining:  
      - Product name  
      - Product short description or description (HTML tags removed, truncated to 300 characters)  
      - Instructions specifying comment tone, style, and variety (neutral, positive, negative, questioning)  
      - Request for conversational tone, occasional brand name inclusion, and brevity  
  - Key expression:  
    ```js
    =={{ `For the product “${$json.name}”, write a natural comment (one sentence). \nDescription: ${($json.short_description || $json.description || '').replace(/<[^>]*>/g,' ').slice(0,300)}` }}
    Comment characteristics:
    - Some neutral, some positive, some negative, some questioning
    - Conversational tone, not formal
    - Occasionally include the brand name in the comment
    - Preferably short
    ```
  - Inputs: Each product JSON from previous node.  
  - Outputs: Passes the prompt string to the AI generation node.  
  - Edge Cases:  
    - Missing `name` or description fields could produce incomplete prompts  
    - HTML stripping could remove important formatting; however, this is intentional.  
  - Version-specific: n8n Set node v3.4  

#### 1.4 AI Review Generation

- **Overview:**  
  Sends the constructed prompt to OpenAI’s GPT-4o-mini model to generate a single-sentence product review based on the prompt.

- **Nodes Involved:**  
  - `Generate Product Comment (AI)`

- **Node Details:**  
  - Type: OpenAI (via n8n LangChain node)  
  - Configuration:  
    - Model: `gpt-4o-mini` (a faster, smaller GPT-4 variant)  
    - Messages: Single message with `content` set from prompt field (`{{$json.prompt}}`)  
    - No additional options set (defaults used)  
  - Credentials: OpenAI API key stored securely in n8n.  
  - Inputs: Receives prompt from “Build Product Comment Prompt” node.  
  - Outputs: AI response JSON including generated message content.  
  - Edge Cases:  
    - API rate limits or quota exceeded  
    - Network timeouts or errors  
    - Unexpected AI output format; fallback extraction provided downstream  
  - Retry: 5 attempts with 2000 ms delay between tries.  
  - Version-specific: Uses n8n LangChain OpenAI node v1.8  

#### 1.5 AI Output Extraction

- **Overview:**  
  Extracts the generated review text from the AI response, accommodating variations in response structure.

- **Nodes Involved:**  
  - `Extract AI Output`

- **Node Details:**  
  - Type: Set  
  - Configuration:  
    - Assigns a new field `review` by extracting from multiple possible AI response paths:  
      - `$json.message?.content`  
      - `$json.choices?.[0]?.message?.content`  
      - `$json.content` (fallback)  
  - Inputs: AI response JSON from the previous node.  
  - Outputs: Passes the clean review text to the posting node.  
  - Edge Cases:  
    - Missing or malformed AI response fields  
    - Empty or nonsensical review text (requires manual review outside workflow)  
  - Version-specific: n8n Set node v3.4  

#### 1.6 Posting Review to WooCommerce

- **Overview:**  
  Posts the generated review as a product comment to the WooCommerce store via the REST API, using randomized guest reviewer names and emails.

- **Nodes Involved:**  
  - `Post Review to Product`

- **Node Details:**  
  - Type: HTTP Request  
  - Configuration:  
    - URL: `https://example.com/wp-json/wc/v3/products/reviews` (replace `example.com`)  
    - Method: POST  
    - Authentication: HTTP Basic Auth using WooCommerce API keys (same as product fetch)  
    - Body parameters include:  
      - `product_id`: from the product JSON id  
      - `review`: from AI-generated review text  
      - `reviewer_email`: generated guest email, unique per review, e.g. `guest+productId-timestamp@example.com`  
      - `reviewer`: random first and last name picked from defined arrays  
    - Timeout: 900,000 ms (15 minutes)  
    - Retry: up to 5 times with 5 seconds delay between attempts  
  - Inputs: Review text and product id from previous nodes.  
  - Outputs: None further; workflow ends here.  
  - Edge Cases:  
    - Authentication failure  
    - API rate limits or rejection of reviews (e.g., if store requires verified purchasers)  
    - Duplicate or spam detection on WooCommerce side  
  - Version-specific: HTTP Request node v4.2  

---

### 3. Summary Table

| Node Name                      | Node Type                          | Functional Role                         | Input Node(s)                 | Output Node(s)                | Sticky Note                                                                                                                           |
|-------------------------------|----------------------------------|---------------------------------------|------------------------------|------------------------------|--------------------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger                   | Workflow start                        | None                         | Fetch Products from WooCommerce | ## Product AI Comment Generator This workflow automatically generates short, natural-sounding product reviews using AI and posts them to your WooCommerce store. It fetches product data via the WooCommerce REST API, crafts a custom AI prompt for each product, and then submits the generated review as a product comment. |
| Fetch Products from WooCommerce | HTTP Request                     | Fetch products from WooCommerce       | When clicking ‘Execute workflow’ | Build Product Comment Prompt  | See note above                                                                                                                       |
| Build Product Comment Prompt    | Set                              | Build AI prompt for review generation | Fetch Products from WooCommerce | Generate Product Comment (AI) | See note above                                                                                                                       |
| Generate Product Comment (AI)   | OpenAI (LangChain)               | Generate AI product review            | Build Product Comment Prompt  | Extract AI Output             | See note above                                                                                                                       |
| Extract AI Output               | Set                              | Extract generated review text         | Generate Product Comment (AI) | Post Review to Product        | See note above                                                                                                                       |
| Post Review to Product          | HTTP Request                     | Post AI review to WooCommerce store  | Extract AI Output             | None                         | See note above                                                                                                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Add node: Manual Trigger  
   - Name: `When clicking ‘Execute workflow’`  
   - No parameters needed.

2. **Create HTTP Request Node to Fetch Products**  
   - Add node: HTTP Request  
   - Name: `Fetch Products from WooCommerce`  
   - Set method to `GET`  
   - URL: `https://example.com/wp-json/wc/v3/products?per_page=100` (replace domain)  
   - Authentication: HTTP Basic Auth  
   - Credentials: Provide WooCommerce API keys in n8n credential manager  
   - Options: Set timeout to 600,000 ms (10 minutes)  
   - Enable batching with batch size 100  
   - Enable retry on fail: 5 retries with 2 seconds wait  
   - Connect input from Manual Trigger node.

3. **Create Set Node for AI Prompt Construction**  
   - Add node: Set  
   - Name: `Build Product Comment Prompt`  
   - Create a new field `prompt` (string) with the value:  
     ```
     =={{ `For the product “${$json.name}”, write a natural comment (one sentence). \nDescription: ${($json.short_description || $json.description || '').replace(/<[^>]*>/g,' ').slice(0,300)}` }}
     Comment characteristics:
     - Some neutral, some positive, some negative, some questioning
     - Conversational tone, not formal
     - Occasionally include the brand name in the comment
     - Preferably short
     ```
   - Connect input from `Fetch Products from WooCommerce`.

4. **Create OpenAI Node for Review Generation**  
   - Add node: OpenAI (LangChain)  
   - Name: `Generate Product Comment (AI)`  
   - Model: `gpt-4o-mini`  
   - Messages: Single message with `content` set to `{{$json.prompt}}`  
   - Credentials: Use OpenAI API key stored in n8n  
   - Enable retry on fail: 5 retries with 2 seconds wait  
   - Connect input from `Build Product Comment Prompt`.

5. **Create Set Node to Extract AI Output**  
   - Add node: Set  
   - Name: `Extract AI Output`  
   - Create a new field `review` (string) with expression:  
     ```
     ={{$json.message?.content || $json.choices?.[0]?.message?.content || $json.content}}
     ```  
   - Connect input from `Generate Product Comment (AI)`.

6. **Create HTTP Request Node to Post Review to WooCommerce**  
   - Add node: HTTP Request  
   - Name: `Post Review to Product`  
   - Set method to `POST`  
   - URL: `https://example.com/wp-json/wc/v3/products/reviews` (replace domain)  
   - Authentication: HTTP Basic Auth using WooCommerce API keys (same credentials as fetch)  
   - Body parameters (JSON or form-data):  
     - `product_id`: `={{ $('Fetch Products from WooCommerce').item.json.id }}`  
     - `review`: `={{ $json.review }}` (or directly from AI output if preferred)  
     - `reviewer_email`:  
       ```
       ={{ `guest+${$('Fetch Products from WooCommerce').item.json.id}-${Math.floor(Date.now()%100000)}@example.com`.toLowerCase() }}
       ```  
     - `reviewer`:  
       ```
       ={{ (() => {
         const first = ['John','Emily','Michael','Sophia','David','Olivia','Daniel','Ava','James','Isabella','Ethan','Mia','William','Charlotte','Benjamin','Amelia','Alexander','Harper','Lucas','Ella'];
         const last  = ['Smith','Johnson','Williams','Brown','Jones','Garcia','Miller','Davis','Rodriguez','Martinez','Hernandez','Lopez','Gonzalez','Wilson','Anderson','Thomas','Taylor','Moore','Jackson','Martin'];
         return `${first[Math.floor(Math.random()*first.length)]} ${last[Math.floor(Math.random()*last.length)]}`;
       })() }}
       ```  
   - Timeout: 900,000 ms (15 minutes)  
   - Retry on fail: 5 retries with 5 seconds wait  
   - Connect input from `Extract AI Output`.

7. **Connect all nodes sequentially:**  
   Manual Trigger → Fetch Products → Build Prompt → Generate AI Review → Extract AI Output → Post Review.

8. **Credential Setup:**  
   - Create WooCommerce HTTP Basic Auth credential with API key and secret.  
   - Create OpenAI API credential with valid API key.

9. **Test Workflow:**  
   - Execute manually, monitor logs and API responses.  
   - Adjust prompt or retry settings as needed.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                      | Context or Link                                                                                          |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| This workflow is designed to generate diverse reviews including neutral, positive, negative, and questioning comments to make product pages appear authentic and engaging.                                                        | Workflow description                                                                                    |
| Replace all placeholder URLs (`https://example.com`) and API credentials with your actual WooCommerce store information before running the workflow.                                                                            | Setup instructions                                                                                       |
| Randomization of reviewer names and emails helps avoid spam detection but does not guarantee bypassing all anti-spam measures on WooCommerce.                                                                                   | Reviewer generation logic                                                                                |
| If your WooCommerce store requires review approval or restricts reviews to verified buyers, this workflow may require further customization to comply with those rules.                                                          | WooCommerce API considerations                                                                           |
| For more details on WooCommerce REST API and review endpoints, see: https://woocommerce.github.io/woocommerce-rest-api-docs/#reviews                                                                                             | WooCommerce API documentation                                                                             |
| OpenAI GPT-4o-mini is a smaller, faster GPT-4 variant ideal for cost-effective text generation. Consider upgrading to larger models for richer content if needed.                                                                | OpenAI model info                                                                                        |
| Workflow designed with idempotency in mind; retries are enabled but ensure API rate limits and quota are monitored to avoid disruptions during bulk processing.                                                                  | Reliability and error handling                                                                           |

---

**Disclaimer:** The provided text originates solely from an automated workflow created with n8n, a tool for integration and automation. This process strictly complies with applicable content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.