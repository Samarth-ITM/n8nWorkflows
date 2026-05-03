Telegram Shopping Assistant: Amazon Product Search with Apify & OpenRouter AI

https://n8nworkflows.xyz/workflows/telegram-shopping-assistant--amazon-product-search-with-apify---openrouter-ai-6319


# Telegram Shopping Assistant: Amazon Product Search with Apify & OpenRouter AI

### 1. Workflow Overview

This workflow implements a **Telegram Shopping Assistant** designed to help users search for Amazon products through Telegram messages. It leverages AI for intent detection and conversational responses, and integrates an Amazon product scraper API via Apify to fetch product details.

**Target Use Cases:**  
- Users query a Telegram bot with product search requests (e.g., "Bluetooth headphones in Amazon")  
- The bot cleans and processes the input, detects if it is a product search or general chat  
- For product queries, it scrapes Amazon product listings (up to 5 items) and returns title, URL, and price  
- For non-product queries, it provides helpful conversational responses about the bot’s purpose

**Logical Blocks:**  
- **1.1 Input Reception:** Receiving Telegram messages  
- **1.2 Input Cleaning:** Normalizing and preparing user input for querying Amazon  
- **1.3 Intent Detection:** Using AI to classify input as product search or general chat  
- **1.4 Routing:** Switching logic based on intent detection to product scraping or chat response  
- **1.5 Product Scraping:** Calling Apify Amazon scraper API and parsing results  
- **1.6 Response Generation:** Formatting and sending responses back to Telegram (product info or chat replies)  
- **1.7 Error Handling & Notes:** Managing invalid inputs and fallback conversational replies

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
Receives incoming Telegram messages via a webhook trigger node.

**Nodes Involved:**  
- Recieve Message

**Node Details:**

- **Recieve Message**  
  - Type: Telegram Trigger (webhook)  
  - Role: Listens for new Telegram messages (only message updates)  
  - Config: Uses Telegram API credentials; webhook configured for message updates  
  - Inputs: Incoming Telegram message payload  
  - Outputs: JSON containing full message object, including text and chat metadata  
  - Edge Cases: Telegram API connectivity issues, webhook misconfiguration, empty/non-text messages

---

#### 2.2 Input Cleaning

**Overview:**  
Normalizes the raw user message by removing platform mentions, punctuation, and encoding it for URL queries.

**Nodes Involved:**  
- Clean userInput

**Node Details:**

- **Clean userInput**  
  - Type: Code (JavaScript)  
  - Role: Processes raw Telegram text input (`message.text`) to extract clean search query  
  - Logic Summary:  
    - Converts input to lowercase  
    - Removes phrases like "in amazon", "on flipkart", etc.  
    - Deletes punctuation except whitespace and word characters  
    - Trims spaces  
    - Validates non-empty query  
    - Encodes spaces as "+" for URL compatibility  
    - Builds Amazon search URL parts  
  - Outputs: JSON with fields: `productQuery` (cleaned text), `amazonSearchPart`, `amazonFullUrl`  
  - Edge Cases: Empty input after cleaning triggers an error JSON response  
  - Failure: Expression errors or empty query lead to early workflow termination or fallback

---

#### 2.3 Intent Detection

**Overview:**  
Uses an AI agent to classify whether the user message is a product search query or a general chat message.

**Nodes Involved:**  
- Intent Detection  
- OpenRouter Chat Model

**Node Details:**

- **OpenRouter Chat Model**  
  - Type: Langchain OpenRouter AI Chat Model  
  - Role: Provides GPT-3.5-turbo-instruct AI model for intent classification  
  - Config: Uses OpenRouter API credentials; model preset "openai/gpt-3.5-turbo-instruct"  
  - Input: Text from cleaned user input node  
  - Output: AI-generated JSON indicating `productQuery` boolean and a `response` text (ignored downstream)  
  - Edge Cases: API auth failure, rate limits, model timeout, malformed responses could break JSON parsing

- **Intent Detection**  
  - Type: Langchain Agent Node  
  - Role: Sends Telegram message text to AI with system prompt to classify intent strictly as product query or not  
  - Prompt instructs AI to reply only with valid JSON, no chit-chat  
  - Input: Raw message text from "Recieve Message" node  
  - Output: JSON with `productQuery` true/false  
  - Edge Cases: AI misclassification, unexpected responses, parsing failures

---

#### 2.4 Routing

**Overview:**  
Directs the workflow flow based on the AI-detected intent to either the product scraping path or the general chat response path.

**Nodes Involved:**  
- Product Vs Response (Switch)

**Node Details:**

- **Product Vs Response**  
  - Type: Switch  
  - Role: Routes flow depending on boolean `productQuery` from the previous Code node parsing AI output  
  - Config: Two outputs: true (product query) → scrape products; false → general response  
  - Inputs: Parsed JSON from Code node (which parsed AI output JSON string)  
  - Outputs:  
    - Output 0: productQuery = true → calls Apify scraper  
    - Output 1: productQuery = false → triggers AI chat response node  
  - Edge Cases: Missing or malformed `productQuery` value may cause routing errors

---

#### 2.5 Product Scraping

**Overview:**  
Sends a POST request to the Apify Amazon scraper API to fetch product details based on the cleaned query.

**Nodes Involved:**  
- Send Post request and get datset items  
- Code (for parsing scraper raw output)  
- Send Amazone Products

**Node Details:**

- **Send Post request and get datset items**  
  - Type: HTTP Request (POST)  
  - Role: Calls Apify API with JSON body containing Amazon search URL (up to 5 items)  
  - Config:  
    - POST method  
    - Body contains `categoryUrls` with Amazon search URL (constructed using cleaned query)  
    - Other parameters: no captcha solver, product details scraped, max 5 items  
  - Inputs: Amazon search part from cleaning node  
  - Outputs: Raw JSON response from Apify scraper API  
  - Edge Cases: API downtime, invalid API URL, rate limits, malformed responses, network timeouts

- **Code**  
  - Type: Code (JavaScript)  
  - Role: Parses the raw JSON string from Apify API (`output` field) into usable JSON object  
  - Input: Raw string from HTTP Request node  
  - Output: JSON array of product items with attributes like title, url, price  
  - Edge Cases: JSON parse errors if API response format changes or is invalid

- **Send Amazone Products**  
  - Type: Telegram node (send message)  
  - Role: Sends each product’s formatted details (Title, URL, Price in ₹) back to the user’s Telegram chat  
  - Config: Text template uses product fields for display  
  - Input: Parsed product JSON array from Code node  
  - Output: None (final output)  
  - Edge Cases: Telegram API failures, message length limits

---

#### 2.6 Response Generation for Non-Product Queries

**Overview:**  
Handles conversational replies when the user’s message is not a product query.

**Nodes Involved:**  
- Response for General  
- OpenRouter Chat Model2  
- Send Response

**Node Details:**

- **Response for General**  
  - Type: Langchain Agent Node  
  - Role: AI agent provides polite, focused conversational replies explaining the bot’s shopping assistant function  
  - Input: Raw user message text  
  - System prompt instructs AI to respond only if not product query, with examples and guidelines  
  - Output: Text response for Telegram message  
  - Edge Cases: AI API errors, ambiguous queries, unexpected output format

- **OpenRouter Chat Model2**  
  - Type: Langchain OpenRouter AI Chat Model  
  - Role: Provides underlying AI model for "Response for General" node  
  - Config: Same OpenRouter GPT-3.5-turbo-instruct model credentials  
  - Edge Cases: Same as above

- **Send Response**  
  - Type: Telegram node (send message)  
  - Role: Sends AI-generated chat response back to Telegram user  
  - Input: Text from "Response for General" node  
  - Edge Cases: Telegram API issues

---

#### 2.7 Notes & Documentation

**Nodes Involved:**  
- Sticky Note

**Node Details:**

- **Sticky Note**  
  - Type: Sticky Note  
  - Content:  
    - Detailed workflow overview  
    - Explanation of flow and key components  
    - Describes input processing, AI integration, product scraping, external dependencies, response format, and error handling  
  - Role: Documentation and quick reference for users/editors

---

### 3. Summary Table

| Node Name                     | Node Type                          | Functional Role                      | Input Node(s)            | Output Node(s)                   | Sticky Note                                                                                                                        |
|-------------------------------|----------------------------------|------------------------------------|-------------------------|---------------------------------|-----------------------------------------------------------------------------------------------------------------------------------|
| Recieve Message                | Telegram Trigger                 | Input Reception - receives messages | None                    | Clean userInput                 | See overall workflow overview in Sticky Note                                                                                      |
| Clean userInput               | Code                            | Cleans and normalizes input text    | Recieve Message          | Intent Detection                | See overall workflow overview in Sticky Note                                                                                      |
| Intent Detection             | Langchain Agent                 | AI intent classification            | Clean userInput          | Code                           | See overall workflow overview in Sticky Note                                                                                      |
| Code                          | Code                            | Parses AI JSON response              | Intent Detection         | Product Vs Response             | See overall workflow overview in Sticky Note                                                                                      |
| Product Vs Response           | Switch                         | Routes based on productQuery boolean | Code                    | Send Post request..., Response for General | See overall workflow overview in Sticky Note                                                                                      |
| Send Post request and get datset items | HTTP Request                   | Calls Apify Amazon scraper API      | Product Vs Response (true) | Code                           | See overall workflow overview in Sticky Note                                                                                      |
| Code                          | Code                            | Parses raw API output                | Send Post request...     | Product Vs Response             | See overall workflow overview in Sticky Note                                                                                      |
| Send Amazone Products          | Telegram                        | Sends product details to Telegram   | Code                     | None                           | See overall workflow overview in Sticky Note                                                                                      |
| Response for General          | Langchain Agent                 | AI conversational replies for non-product queries | Product Vs Response (false) | Send Response                 | See overall workflow overview in Sticky Note                                                                                      |
| OpenRouter Chat Model         | Langchain AI Model              | Provides AI model for intent detection | None                    | Intent Detection                | See overall workflow overview in Sticky Note                                                                                      |
| OpenRouter Chat Model2        | Langchain AI Model              | Provides AI model for general response | None                    | Response for General            | See overall workflow overview in Sticky Note                                                                                      |
| Send Response                 | Telegram                        | Sends AI chat responses             | Response for General     | None                           | See overall workflow overview in Sticky Note                                                                                      |
| Sticky Note                  | Sticky Note                    | Documentation                       | None                     | None                           | ## Overview\nTelegram bot that scrapes Amazon products using n8n automation. Users send product queries via Telegram and receive product details (title, URL, price).\n\n## Workflow Flow\nTelegram Trigger → Receives messages\nClean Input → Normalizes user text (removes "in amazon", punctuation, etc.)\nIntent Detection → AI determines if message is product search or chat\nSwitch → Routes to appropriate handler\nProduct Route → Calls Apify Amazon scraper API (max 5 items)\nChat Route → AI conversational response\nSend Response → Formatted message back to Telegram\n\n## Key Components:-\n\n  **Input Processing**\n  - Removes platform mentions ("in amazon", "on flipkart")\n- Cleans punctuation and normalizes text\n- URL-encodes for search queries\n\n**AI Integration**\n- Uses OpenRouter API with GPT-3.5-turbo model\n- Intent classification (product vs conversation)\n- Shopping assistant responses\n\n**Product Scraping**\n- Apify Amazon scraper integration\n- Returns: title, URL, price in Indian Rupees\n- Limited to 5 products per query\n\n**External Dependencies**\n- Telegram Bot API\n- OpenRouter API (AI models)\n- Apify Amazon Scraper API\n\n**Response Format**\n- Title: [Product Name]\n- URL: [Amazon Link]\n- Price: ₹[Amount]\n\n**Error Handling**\n- Validates empty queries\n- Fallback responses for unrecognized intents |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node ("Recieve Message")**  
   - Node Type: Telegram Trigger  
   - Configure with your Telegram Bot API credentials  
   - Set to listen to `message` updates only  
   - No additional fields required

2. **Create Code Node ("Clean userInput")**  
   - Node Type: Code (JavaScript)  
   - Input: `message.text` from Telegram trigger  
   - Code:  
     - Lowercase the input  
     - Remove phrases like "in amazon", "on flipkart", etc.  
     - Remove punctuation except word characters and spaces  
     - Trim spaces  
     - Validate non-empty result; if empty, return error JSON  
     - Encode spaces as "+" for URL  
     - Build `amazonSearchPart` and `amazonFullUrl` fields  
   - Output JSON with cleaned query and URL parts

3. **Create Langchain OpenRouter AI Node ("OpenRouter Chat Model")**  
   - Node Type: Langchain Chat OpenRouter  
   - Credentials: OpenRouter API account  
   - Model preset: `openai/gpt-3.5-turbo-instruct`  
   - No additional options needed

4. **Create Langchain Agent Node ("Intent Detection")**  
   - Node Type: Langchain Agent  
   - Input text: Expression from `$('Recieve Message').item.json.message.text`  
   - System prompt: instruct AI to classify if message is product search or general chat, reply only with JSON containing `productQuery` boolean  
   - Prompt language: English  
   - Connect AI model input to "OpenRouter Chat Model" node

5. **Create Code Node ("Code")**  
   - Node Type: Code (JavaScript)  
   - Purpose: Parse the JSON string output returned by the AI agent in the previous node  
   - Input: `output` field containing JSON string from AI  
   - Output: Parsed JSON object with `productQuery` boolean and `response` string

6. **Create Switch Node ("Product Vs Response")**  
   - Node Type: Switch  
   - Condition: Check if `productQuery` equals `true` (boolean)  
   - Output 0: For true, route to product scraping  
   - Output 1: For false, route to general response

7. **Create HTTP Request Node ("Send Post request and get datset items")**  
   - Node Type: HTTP Request (POST)  
   - URL: Apify Amazon Scraper API endpoint (use correct URL)  
   - Method: POST  
   - Body (JSON):  
     ```json
     {
       "categoryUrls": [
         {
           "url": "https://www.amazon.in/s?k={{ $('Clean userInput').item.json.amazonSearchPart }}",
           "method": "GET"
         }
       ],
       "ensureLoadedProductDescriptionFields": false,
       "maxItemsPerStartUrl": 5,
       "scrapeProductDetails": true,
       "scrapeProductVariantPrices": false,
       "useCaptchaSolver": false
     }
     ```
   - Send body as JSON

8. **Create Code Node ("Code")**  
   - Node Type: Code (JavaScript)  
   - Purpose: Parse the `output` field from the HTTP request response (which is a raw JSON string) into a JSON object  
   - Return parsed JSON array of products

9. **Create Telegram Node ("Send Amazone Products")**  
   - Node Type: Telegram  
   - Chat ID: From original message `$('Recieve Message').item.json.message.chat.id`  
   - Text: Template string showing product title, URL, and price in ₹ (rupees)  
   - No attribution appended

10. **Create Langchain OpenRouter AI Node ("OpenRouter Chat Model2")**  
    - Same setup as step 3, dedicated for general chat responses

11. **Create Langchain Agent Node ("Response for General")**  
    - Input text: `$('Recieve Message').item.json.message.text`  
    - System prompt: AI should provide helpful, concise conversational replies only when input is not a product query, explain bot capabilities  
    - Connect AI model input to "OpenRouter Chat Model2"

12. **Create Telegram Node ("Send Response")**  
    - Chat ID: From original message  
    - Text: AI-generated response text from "Response for General" node  
    - No attribution appended

13. **Connect nodes:**  
    - "Recieve Message" → "Clean userInput" → "Intent Detection" → "Code" (parse AI intent) → "Product Vs Response"  
    - "Product Vs Response" true → "Send Post request and get datset items" → "Code" (parse API output) → "Send Amazone Products"  
    - "Product Vs Response" false → "Response for General" → "Send Response"

14. **Add Sticky Note with workflow overview and instructions**

15. **Set credentials:**  
    - Telegram API credentials (Bot Token) in Telegram Nodes and Trigger  
    - OpenRouter API credentials in Langchain nodes  
    - Ensure proper API keys for Apify Amazon Scraper (if required, configure in HTTP Request node)

16. **Test thoroughly:**  
    - Test product queries with various phrases and check product results  
    - Test greetings and non-product messages and verify AI conversational replies  
    - Handle empty or invalid inputs gracefully

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                       | Context or Link                                  |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------|
| Telegram bot that scrapes Amazon products using n8n automation. Users send product queries via Telegram and receive product details (title, URL, price). Workflow flow: Telegram Trigger → Clean Input → Intent Detection → Switch → Product Scraping or Chat Response → Send Response | Sticky Note content embedded in workflow JSON   |
| Uses OpenRouter API (GPT-3.5-turbo-instruct) for AI intent classification and conversational responses.                                                                                                                                                            | OpenRouter API documentation                     |
| Integrates Apify Amazon Scraper API for live product data (max 5 items per query), returning title, URL, and price in Indian Rupees (₹).                                                                                                                           | Apify Amazon Scraper API documentation           |
| Input cleaning removes platform mentions ("in amazon", "on flipkart"), punctuation, and normalizes text to build search queries compatible with Amazon URLs.                                                                                                      | N8N Code node script in "Clean userInput" node  |
| Error handling includes validation for empty queries and fallback AI responses for unrecognized intents.                                                                                                                                                           | Built into code and AI prompt logic               |
| Telegram API credentials and OpenRouter API credentials must be properly configured for the workflow to function.                                                                                                                                                   | Credentials configuration in n8n                  |

---

**Disclaimer:** The text provided is extracted exclusively from an automated n8n workflow. It strictly respects all content policies and contains no illegal or protected elements. All processed data is legal and publicly available.