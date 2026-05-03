Automated Product Inquiry Responder with GPT-4 and Google Sheets

https://n8nworkflows.xyz/workflows/automated-product-inquiry-responder-with-gpt-4-and-google-sheets-5809


# Automated Product Inquiry Responder with GPT-4 and Google Sheets

### 1. Workflow Overview

This workflow automates responding to customer product inquiries (specifically shoe orders) via an HTTP webhook interface. It listens for incoming messages containing product requests, uses AI (OpenAI GPT models) to extract relevant product details, matches those details against an inventory stored in Google Sheets, and then generates a friendly, informative reply including product availability, pricing, and suggestions. The key logical blocks are:

- **1.1 Input Reception:** Receiving customer inquiries via webhook POST requests.  
- **1.2 AI Parsing:** Using GPT to extract structured product attributes (brand, model, size, color) from free-text messages.  
- **1.3 Data Extraction:** Parsing the AI output JSON to usable variables.  
- **1.4 Product Lookup:** Loading full product inventory from Google Sheets and filtering it by extracted attributes.  
- **1.5 AI Response Generation:** Using GPT to craft a customer-friendly response based on filtered product data.  
- **1.6 Response Delivery:** Sending the AI-generated message back as the webhook response.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
Receives incoming customer messages via a POST webhook on `/shoe-orders`. This is the entry point to the workflow.

- **Nodes Involved:**  
`Receive Request`

- **Node Details:**  
  - **Receive Request**  
    - Type: Webhook  
    - Role: Accepts POST requests at `/shoe-orders` path. It waits for a JSON body expected to contain a `message` field with customer inquiry text.  
    - Config: HTTP method POST, response mode configured to respond with output node data.  
    - Inputs: External HTTP client (e.g., chatbot, website form)  
    - Outputs: JSON with request body forwarded downstream  
    - Failure modes: Missing or malformed HTTP requests, invalid JSON payloads  
    - Requires webhook URL exposure and proper security if public-facing  

#### 2.2 AI Parsing

- **Overview:**  
Uses an OpenAI GPT-3.5-turbo model to parse the incoming free-text message into structured JSON with shoe details.

- **Nodes Involved:**  
`Parse Request AI`

- **Node Details:**  
  - **Parse Request AI**  
    - Type: Langchain Chain LLM node  
    - Role: Extracts shoe details (brand, model, size, color) from the message text into a strict JSON format.  
    - Config: System prompt instructs the AI to return only valid JSON with specified fields. Examples provided to guide extraction.  
    - Input: Message text extracted from webhook body.  
    - Output: AI response containing JSON string with shoe attributes.  
    - Expressions: `={{ $('Receive Request').first().json.body.message }}` to pass message text.  
    - Failure modes: AI output not parseable JSON, incomplete extraction, API rate limits or errors.  
    - Uses OpenAI GPT-3.5-turbo model with connected OpenAI API credentials.

#### 2.3 Data Extraction

- **Overview:**  
Parses the JSON string returned from AI into usable JSON fields for downstream filtering.

- **Nodes Involved:**  
`Extract Data`

- **Node Details:**  
  - **Extract Data**  
    - Type: Code node (JavaScript)  
    - Role: Parses AI JSON output string and outputs separate properties: brand, model, size, color.  
    - Config: Parses `input.text` JSON, returns object with fallback empty strings if fields missing.  
    - Input: AI response from `Parse Request AI`.  
    - Output: Clean JSON with extracted product attributes.  
    - Failure modes: JSON parse errors if AI output malformed, runtime JS errors.

#### 2.4 Product Lookup

- **Overview:**  
Retrieves full product inventory from Google Sheets and filters it based on extracted attributes.

- **Nodes Involved:**  
`Product Database`, `Filter Products`

- **Node Details:**  
  - **Product Database**  
    - Type: Google Sheets node  
    - Role: Reads all rows from a specific Google Sheet (inventory list).  
    - Config: Spreadsheet ID and sheet name (gid=0) specified; linked to Google Sheets OAuth2 credentials.  
    - Input: Triggered by `Extract Data` output  
    - Output: All product rows as JSON objects with fields like Brand, Model, Size, Color, Price, Stock Quantity.  
    - Failure modes: Google API auth failures, rate limits, empty or unreachable sheet.  
  - **Filter Products**  
    - Type: Code node (JavaScript)  
    - Role: Filters the full inventory array to those matching brand, model, size, color extracted earlier.  
    - Config: Case insensitive string comparisons for brand, model, color; exact match for size. Model and color optional matches.  
    - Input: Entire product list from `Product Database` and extracted search parameters from `Extract Data`.  
    - Output: Filtered product list matching query.  
    - Failure modes: Empty product list, no matches found, field naming mismatches.

#### 2.5 AI Response Generation

- **Overview:**  
Uses GPT-4 to generate a customer-facing reply including availability, pricing, and suggestions based on filtered products.

- **Nodes Involved:**  
`AI Manager`

- **Node Details:**  
  - **AI Manager**  
    - Type: Langchain Chain LLM node  
    - Role: Generates a friendly, helpful message using GPT-4 based on product data and original customer message.  
    - Config: System prompt instructs the AI to behave as a shoe store assistant, including price, stock quantity, and alternative suggestions if out of stock.  
    - Input: Original customer message (from webhook) and filtered product data (passed as JSON string).  
    - Output: Text response to be sent back to customer.  
    - Failure modes: AI model errors, timeouts, insufficient product data, API limits.  
    - Uses GPT-4 model with OpenAI credentials.

#### 2.6 Response Delivery

- **Overview:**  
Sends the AI-generated text response back via webhook HTTP response.

- **Nodes Involved:**  
`Send Response`

- **Node Details:**  
  - **Send Response**  
    - Type: Respond to Webhook node  
    - Role: Sends the AI-generated friendly message text as the response to the original HTTP request.  
    - Config: Responds with plain text body containing AI reply (`{{$json.text}}`).  
    - Input: Output from `AI Manager` node.  
    - Output: HTTP response to client.  
    - Failure modes: Network errors, malformed response text.

---

### 3. Summary Table

| Node Name         | Node Type                          | Functional Role                          | Input Node(s)      | Output Node(s)       | Sticky Note                                                                                 |
|-------------------|----------------------------------|----------------------------------------|--------------------|----------------------|---------------------------------------------------------------------------------------------|
| Receive Request   | Webhook                          | Receive incoming customer message      | External HTTP call  | Parse Request AI      | 📥 Incoming customer message                                                                |
| Parse Request AI  | Langchain Chain LLM (GPT-3.5)   | Extract shoe details as JSON            | Receive Request     | Extract Data          | 🤖 Extract brand/model/size from text                                                       |
| Extract Data      | Code                            | Parse AI JSON output into fields        | Parse Request AI    | Product Database      | 📊 Process JSON output                                                                      |
| Product Database  | Google Sheets                   | Load product inventory                   | Extract Data        | Filter Products       | 📋 Load all products from inventory                                                        |
| Filter Products   | Code                            | Filter inventory by extracted attributes| Product Database    | AI Manager            | 🔍 Find matching items only                                                                 |
| AI Manager        | Langchain Chain LLM (GPT-4)    | Generate friendly response text          | Filter Products     | Send Response         | 💬 Generate friendly response                                                              |
| Send Response     | Respond to Webhook              | Send AI reply as HTTP response           | AI Manager         | HTTP client           | 📤 Deliver result to customer                                                               |
| Sticky Note       | Sticky Note                    | Workflow overview comment                 | -                  | -                    | Workflow Node Descriptions: Receive Request, Parse Request AI, Extract Data, Product Database, Filter Products, AI Manager, Send Response |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node: "Receive Request"**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `/shoe-orders`  
   - Response Mode: `Response Node` (to reply via downstream node)  
   - No authentication configured (add if public exposure required)

2. **Add Langchain Chain LLM Node: "Parse Request AI"**  
   - Model: GPT-3.5-turbo  
   - Input Text Expression: `={{ $('Receive Request').first().json.body.message }}`  
   - Prompt: System message instructing extraction of shoe details into JSON with fields brand, model, size, color. Include examples for clarity.  
   - Set `hasOutputParser` to true to enable JSON parsing.  
   - Connect input from `Receive Request`.  
   - Configure OpenAI API credentials.

3. **Add Code Node: "Extract Data"**  
   - JavaScript to parse `input.text` JSON string into `{brand, model, size, color}` fields.  
   - Sample code snippet:  
     ```js
     const input = $input.first().json;
     const parsedData = JSON.parse(input.text);
     return [{
       json: {
         brand: parsedData.brand || "",
         model: parsedData.model || "",
         size: parsedData.size || "",
         color: parsedData.color || ""
       }
     }];
     ```  
   - Connect from `Parse Request AI`.

4. **Add Google Sheets Node: "Product Database"**  
   - Operation: Read rows from sheet  
   - Spreadsheet ID: (Your Google Sheet ID containing product inventory)  
   - Sheet Name: `gid=0` or your default sheet name  
   - Connect from `Extract Data`.  
   - Configure Google Sheets OAuth2 credentials.

5. **Add Code Node: "Filter Products"**  
   - JavaScript code to filter the full product list based on brand, model, size, and color extracted earlier.  
   - Use case-insensitive comparison for brand, model, color; exact size match.  
   - Example logic:  
     ```js
     const allItems = $input.all();
     const search = $('Extract Data').first().json;
     const filtered = allItems.filter(item => {
       const brandMatch = item.json.Brand?.toLowerCase() === search.brand?.toLowerCase();
       const modelMatch = search.model ? item.json.Model?.toLowerCase().includes(search.model?.toLowerCase()) : true;
       const sizeMatch = item.json.Size?.toString() === search.size;
       const colorMatch = search.color ? item.json.Color?.toLowerCase().includes(search.color?.toLowerCase()) : true;
       return brandMatch && modelMatch && sizeMatch && colorMatch;
     });
     return filtered;
     ```  
   - Connect from `Product Database`.

6. **Add Langchain Chain LLM Node: "AI Manager"**  
   - Model: GPT-4  
   - Input Text Expression: `={{ $('Receive Request').first().json.body.message }}`  
   - System prompt: Act as a friendly shoe store assistant. Use product data JSON (`{{ JSON.stringify($input.all()) }}`) to generate a helpful message including price, availability, stock quantity, and alternatives if out of stock.  
   - Connect from `Filter Products`.  
   - Configure OpenAI API credentials.

7. **Add Respond to Webhook Node: "Send Response"**  
   - Respond With: Text  
   - Response Body: `={{ $json.text }}` (output from `AI Manager`)  
   - Connect from `AI Manager`.

8. **Connect Nodes in Order:**  
   - `Receive Request` → `Parse Request AI` → `Extract Data` → `Product Database` → `Filter Products` → `AI Manager` → `Send Response`

9. **Credentials Setup:**  
   - OpenAI API credentials configured for GPT-3.5 and GPT-4 models.  
   - Google Sheets OAuth2 credentials configured with access to the inventory spreadsheet.

10. **Testing & Validation:**  
    - Send POST request with JSON body `{ "message": "Do you have Nike Air Max size 40?" }` to webhook URL.  
    - Verify parsed extraction, product filtering, and friendly AI response.  
    - Handle errors such as no matches found or malformed input.

---

### 5. General Notes & Resources

| Note Content                                                                                                         | Context or Link                                                                                     |
|----------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| Workflow nodes represent a clear logical flow from receiving input to AI-powered parsing, filtering, and responding. | Overview provided in a large sticky note inside the workflow (see node "Sticky Note").             |
| Use of two different OpenAI models: GPT-3.5-turbo for parsing input and GPT-4 for response generation.                | Balances cost-efficiency and quality in AI processing.                                            |
| Google Sheets serves as the dynamic product inventory source, enabling easy updates without code changes.              | Sheet URL: https://docs.google.com/spreadsheets/d/1AczF58510i-QEppNi_UnzgqqfX0ickNJZjFW2Bu-6mI    |
| The workflow assumes structured columns in the sheet named Brand, Model, Size, Color, Price, Stock Quantity.          | Data schema must be consistent for filtering to work correctly.                                   |
| Ensure webhook endpoint security if deploying publicly; consider adding authentication or IP whitelisting.            | Important for production deployments.                                                             |
| Error handling is minimal in nodes; consider adding catch nodes or error workflows for API failures or parsing issues. | For robustness in production environments.                                                        |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.