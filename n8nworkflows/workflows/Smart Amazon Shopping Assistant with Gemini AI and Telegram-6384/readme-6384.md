Smart Amazon Shopping Assistant with Gemini AI and Telegram

https://n8nworkflows.xyz/workflows/smart-amazon-shopping-assistant-with-gemini-ai-and-telegram-6384


# Smart Amazon Shopping Assistant with Gemini AI and Telegram

### 1. Workflow Overview

This workflow implements a **Smart Amazon Shopping Assistant** integrated with **Google Gemini AI** and **Telegram**. It is designed to receive user messages via Telegram and intelligently determine whether the user seeks product-related information (shopping queries) or general chat interaction. For product-related queries, it scrapes Amazon product data, processes and aggregates it, then uses AI to analyze and recommend the best products. For non-product queries, it generates conversational responses clarifying its capabilities.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Receives Telegram messages and triggers the workflow.
- **1.2 Query Preprocessing:** Cleans and normalizes user input for searching.
- **1.3 Intent Classification:** AI-based classification to determine if the message is product-related.
- **1.4 Routing Decision:** Routes the workflow either to product search or chat response path.
- **1.5 Chat Response Generation:** Creates AI-generated conversational replies for non-shopping queries.
- **1.6 Product Search:** Calls Apify's Amazon scraper API to fetch product listings.
- **1.7 Product Data Processing:** Extracts and standardizes relevant product information.
- **1.8 Data Aggregation:** Combines all product data into a single dataset.
- **1.9 Product Recommendation Engine:** AI analyzes products and user query to generate shopping recommendations.
- **1.10 Output Delivery:** Sends either the chat response or product recommendations back to Telegram.
- **1.11 AI Models Integration:** Utilizes multiple Google Gemini AI models specialized for chat, intent detection, and product analysis.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** Listens for incoming Telegram messages to trigger the workflow.
- **Nodes Involved:**  
  - Telegram Message Receiver
- **Node Details:**

  - **Telegram Message Receiver**  
    - Type: Telegram Trigger  
    - Role: Listens for new Telegram messages (`message` update type).  
    - Configuration: Uses a Telegram API credential. Webhook ID set for receiving messages.  
    - Inputs: External Telegram webhook messages.  
    - Outputs: Passes raw message JSON downstream.  
    - Edge Cases: Telegram API downtime, webhook misconfiguration, invalid message format.

#### 2.2 Query Preprocessing

- **Overview:** Cleans user message text by removing platform mentions and punctuation, preparing a formatted query for scraping.
- **Nodes Involved:**  
  - Product Query Cleaner
- **Node Details:**

  - **Product Query Cleaner**  
    - Type: Code node (JavaScript)  
    - Role:  
      - Converts input text to lowercase.  
      - Removes phrases like "in Amazon", "on Flipkart", etc.  
      - Strips punctuation.  
      - Trims whitespace.  
      - Encodes spaces as plus signs for URL use.  
      - Constructs Amazon search URL snippet and full URL.  
    - Key Variables: `cleanedInput`, `amazonSearchPart`, `amazonFullUrl`  
    - Inputs: JSON with raw Telegram message text.  
    - Outputs: JSON containing cleaned product query and URL parts.  
    - Edge Cases: Empty or invalid input after cleaning; returns error JSON in that case.

#### 2.3 Intent Classification

- **Overview:** Uses AI to classify whether the message is related to product search or casual chat.
- **Nodes Involved:**  
  - Message Intent Classifier  
  - Structured Output Parser  
  - Google Gemini Chat Model2
- **Node Details:**

  - **Google Gemini Chat Model2**  
    - Type: AI Language Model (Google Gemini)  
    - Role: Processes classification prompt.  
    - Configuration: Uses Google Palm API credentials.  
    - Inputs: Raw message text from Telegram.  
    - Outputs: AI response with classification JSON.  
    - Edge Cases: API authentication failure, timeout, unexpected AI output format.

  - **Structured Output Parser**  
    - Type: AI Output Parser (Structured JSON)  
    - Role: Parses AI response into structured JSON with `productQuery` boolean and `response` text.  
    - Inputs: AI output from Gemini Chat Model2.  
    - Outputs: Parsed JSON for routing logic.  
    - Edge Cases: Parsing error if AI output is malformed.

  - **Message Intent Classifier**  
    - Type: AI Agent  
    - Role: Combines AI model and parser for intent decision.  
    - Inputs: Cleaned message text from Product Query Cleaner.  
    - Outputs: Structured JSON indicating product query or not.  
    - Connections: Outputs to Product vs Chat Router.

#### 2.4 Routing Decision

- **Overview:** Routes workflow based on whether the input is product-related or general chat.
- **Nodes Involved:**  
  - Product vs Chat Router
- **Node Details:**

  - **Product vs Chat Router**  
    - Type: Switch node  
    - Role: Checks `productQuery` boolean from Intent Classifier output.  
    - Configuration: Two outputs — true (product query), false (chat).  
    - Inputs: Structured JSON with `productQuery`.  
    - Outputs: Routes to Amazon Product Scraper or Chat Response Generator.  
    - Edge Cases: Missing or invalid `productQuery` value.

#### 2.5 Chat Response Generation

- **Overview:** Generates AI-powered friendly chat responses for non-product queries.
- **Nodes Involved:**  
  - Google Gemini Chat Model1  
  - Chat Response Generator  
  - Send Chat Response
- **Node Details:**

  - **Google Gemini Chat Model1**  
    - Type: AI Language Model  
    - Role: Generates conversational replies for casual chat messages.  
    - Credentials: Google Palm API.  
    - Inputs: Raw Telegram message text.  
    - Outputs: AI-generated response text.

  - **Chat Response Generator**  
    - Type: AI Agent  
    - Role: Defines system message guiding AI to respond helpfully but briefly to non-product queries.  
    - Inputs: Telegram message text.  
    - Outputs: Chat response text.

  - **Send Chat Response**  
    - Type: Telegram node  
    - Role: Sends AI chat reply back to the originating Telegram chat.  
    - Configuration: Uses Telegram API credentials.  
    - Inputs: AI response text, and chat ID from incoming message.  
    - Outputs: Sends message to user.  
    - Edge Cases: Telegram API errors, invalid chat IDs.

#### 2.6 Product Search

- **Overview:** Invokes Apify's Amazon product scraper API to fetch product listings based on cleaned query.
- **Nodes Involved:**  
  - Amazon Product Scraper
- **Node Details:**

  - **Amazon Product Scraper**  
    - Type: HTTP Request  
    - Role: Posts request to Apify API with Amazon search URL constructed from cleaned query.  
    - Configuration:  
      - POST method with JSON body specifying category URL (Amazon search), max 10 items, product details to scrape.  
      - Token parameter for Apify API authentication (placeholder "value" to be replaced).  
    - Inputs: Cleaned query from Product Query Cleaner.  
    - Outputs: Raw scraped product data JSON array.  
    - Edge Cases: Network errors, API token invalid or expired, empty results.

#### 2.7 Product Data Processing

- **Overview:** Extracts and standardizes product information from raw scraped data.
- **Nodes Involved:**  
  - Product Data Processor
- **Node Details:**

  - **Product Data Processor**  
    - Type: Code node (JavaScript)  
    - Role:  
      - Iterates over each scraped product.  
      - Extracts title, price, star rating, total reviews (calculated), product URL, description.  
      - Calculates total reviews by summing star breakdown percentages.  
      - Returns an array of simplified product JSON objects.  
    - Inputs: Raw product data from Amazon Product Scraper.  
    - Outputs: Cleaned and structured product list JSON.  
    - Edge Cases: Missing or malformed product fields; defaults applied (e.g., 'N/A', 0).

#### 2.8 Data Aggregation

- **Overview:** Combines all processed product items into a single dataset for AI analysis.
- **Nodes Involved:**  
  - Product List Combiner
- **Node Details:**

  - **Product List Combiner**  
    - Type: Aggregate node  
    - Role: Aggregates all individual product JSON objects into one combined array.  
    - Inputs: Multiple product JSON items from Product Data Processor.  
    - Outputs: Single aggregated JSON dataset for recommendation engine.  
    - Edge Cases: Empty input array.

#### 2.9 Product Recommendation Engine

- **Overview:** Uses AI to analyze aggregated product data and user query, producing top product recommendations formatted for Telegram.
- **Nodes Involved:**  
  - Google Gemini Chat Model3  
  - Product Recommendation Engine
- **Node Details:**

  - **Google Gemini Chat Model3**  
    - Type: AI Language Model  
    - Role: Runs AI analysis and recommendation prompt.  
    - Credentials: Google Palm API.  
    - Inputs: User message text plus aggregated product data JSON.  
    - Outputs: AI-generated recommendation text formatted for Telegram.  
    - Edge Cases: API failures, large input size causing truncation.

  - **Product Recommendation Engine**  
    - Type: AI Agent  
    - Role: Defines detailed system prompt for analyzing product features, pricing, user needs, and generating concise Telegram-ready recommendations with strict character limits and formatting rules.  
    - Inputs: Combined product dataset and user message text.  
    - Outputs: Telegram-formatted product recommendation message.

#### 2.10 Output Delivery

- **Overview:** Sends AI-generated product recommendations to the Telegram user.
- **Nodes Involved:**  
  - Send Product Recommendations
- **Node Details:**

  - **Send Product Recommendations**  
    - Type: Telegram node  
    - Role: Sends product recommendation text message back to the Telegram chat.  
    - Configuration: Uses Telegram API credentials; targets the originating chat ID.  
    - Inputs: AI recommendation message from Product Recommendation Engine.  
    - Outputs: Message delivery on Telegram.  
    - Edge Cases: Telegram message length limits, API errors.

#### 2.11 AI Models Integration

- **Overview:** Three Google Gemini AI models are used for distinct purposes: chat response, intent classification, and product analysis.
- **Nodes Involved:**  
  - Google Gemini Chat Model1 (Chat responses)  
  - Google Gemini Chat Model2 (Intent classification)  
  - Google Gemini Chat Model3 (Product recommendation)
- **Node Details:**

  - All use Google Palm API credentials with separate nodes for modularity and specialized prompt contexts.  
  - Enable consistent AI experience across the workflow.  
  - Edge Cases: API quota limits, credential misconfiguration.

---

### 3. Summary Table

| Node Name                    | Node Type                             | Functional Role                      | Input Node(s)                  | Output Node(s)                          | Sticky Note                                                                                 |
|------------------------------|-------------------------------------|------------------------------------|-------------------------------|----------------------------------------|--------------------------------------------------------------------------------------------|
| Telegram Message Receiver     | Telegram Trigger                    | Receives Telegram messages          | —                             | Product Query Cleaner                   | Message Entry Point: Receives all incoming Telegram messages; webhook activation             |
| Product Query Cleaner         | Code                               | Cleans and formats product query    | Telegram Message Receiver      | Message Intent Classifier               | Query Cleanup & Preparation: Removes platform mentions, punctuation; prepares query         |
| Google Gemini Chat Model2     | AI Language Model (Google Gemini)  | AI model for intent classification  | —                             | Message Intent Classifier (ai_languageModel) | Google Gemini Integration: One of three specialized AI models                              |
| Structured Output Parser      | AI Output Parser (Structured JSON) | Parses AI JSON classification output | Google Gemini Chat Model2      | Message Intent Classifier (ai_outputParser) |                                                                                             |
| Message Intent Classifier     | AI Agent                          | Determines if message is product query | Product Query Cleaner, Structured Output Parser | Product vs Chat Router                 | Smart Message Routing: AI separates shopping from casual chat                              |
| Product vs Chat Router        | Switch                            | Routes workflow to product or chat  | Message Intent Classifier      | Amazon Product Scraper; Chat Response Generator | Workflow Split Point: Routes based on AI classification                                   |
| Amazon Product Scraper        | HTTP Request                      | Scrapes Amazon product data         | Product vs Chat Router         | Product Data Processor                  | Amazon Product Discovery: Scrapes top 10 products from Amazon                             |
| Product Data Processor        | Code                             | Processes raw product data          | Amazon Product Scraper         | Product List Combiner                   | Product Data Transformation: Extracts key product info                                   |
| Product List Combiner         | Aggregate                        | Combines all product data           | Product Data Processor         | Product Recommendation Engine          | Combine All Products: Merges product data for AI analysis                               |
| Google Gemini Chat Model3     | AI Language Model (Google Gemini) | AI for product recommendation       | —                             | Product Recommendation Engine (ai_languageModel) | Google Gemini Integration                                                               |
| Product Recommendation Engine | AI Agent                        | Generates AI product recommendations | Product List Combiner          | Send Product Recommendations            | Smart Product Analysis: AI ranks and formats top products                                |
| Send Product Recommendations | Telegram Node                    | Sends recommendations to Telegram   | Product Recommendation Engine | —                                      | Message Delivery: Sends shopping results to Telegram                                   |
| Google Gemini Chat Model1     | AI Language Model (Google Gemini) | AI for chat response generation     | —                             | Chat Response Generator (ai_languageModel) | Google Gemini Integration                                                               |
| Chat Response Generator       | AI Agent                        | Generates chat replies for non-product queries | Product vs Chat Router         | Send Chat Response                      | Non-Product Conversations: AI creates helpful chat replies                              |
| Send Chat Response            | Telegram Node                    | Sends chat replies to Telegram      | Chat Response Generator        | —                                      | Message Delivery: Sends chat messages to Telegram                                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Message Receiver node:**
   - Type: Telegram Trigger  
   - Configure Telegram API credentials with your bot token.  
   - Set to listen for `message` updates.  
   - This node triggers the workflow on incoming messages.

2. **Create Product Query Cleaner node (Code):**
   - Connect Telegram Message Receiver → Product Query Cleaner.  
   - Add JavaScript code to:  
     - Normalize input to lowercase.  
     - Remove phrases like "in Amazon", "on Flipkart".  
     - Strip punctuation.  
     - Trim spaces.  
     - Encode spaces as plus signs.  
     - Construct Amazon search URL parts.  
   - Outputs cleaned query JSON or error if input invalid.

3. **Create Google Gemini Chat Model2 node:**
   - Type: AI Language Model (Google Gemini)  
   - Connect Product Query Cleaner → Message Intent Classifier (via ai_languageModel).  
   - Use Google Palm API credential.  
   - No special prompt here (handled in agent).

4. **Create Structured Output Parser node:**
   - Type: AI Output Parser (Structured JSON)  
   - Connect Google Gemini Chat Model2 → Structured Output Parser (ai_outputParser).  
   - Provide JSON schema example with `productQuery` (boolean) and `response` (string).

5. **Create Message Intent Classifier node (AI Agent):**
   - Connect Product Query Cleaner → Message Intent Classifier (main).  
   - Connect Structured Output Parser → Message Intent Classifier (ai_outputParser).  
   - Connect Google Gemini Chat Model2 → Message Intent Classifier (ai_languageModel).  
   - Configure system prompt to classify message as product query or not, returning JSON only.

6. **Create Product vs Chat Router node (Switch):**
   - Connect Message Intent Classifier → Product vs Chat Router.  
   - Configure switch rules:  
     - Output 1 if `productQuery` is true.  
     - Output 2 if `productQuery` is false.

7. **Configure Chat Response Path:**

   7.1. **Create Google Gemini Chat Model1 node:**
        - Type: AI Language Model (Google Gemini)  
        - Connect Product vs Chat Router (false output) → Chat Response Generator (ai_languageModel).  
        - Use Google Palm API credentials.

   7.2. **Create Chat Response Generator node (AI Agent):**
        - Connect Product vs Chat Router (false output) → Chat Response Generator (main).  
        - Connect Google Gemini Chat Model1 → Chat Response Generator (ai_languageModel).  
        - Configure system prompt to generate brief, helpful chat responses for non-shopping queries.

   7.3. **Create Send Chat Response node:**
        - Type: Telegram node.  
        - Connect Chat Response Generator → Send Chat Response.  
        - Use Telegram API credentials.  
        - Set `chatId` to incoming message chat ID.  
        - Send AI response text.

8. **Configure Product Search Path:**

   8.1. **Create Amazon Product Scraper node (HTTP Request):**
        - Connect Product vs Chat Router (true output) → Amazon Product Scraper.  
        - Method: POST  
        - URL: `https://api.apify.com/v2/acts/junglee~free-amazon-product-scraper/run-sync-get-dataset-items?token=YOUR_TOKEN` (replace YOUR_TOKEN)  
        - JSON Body:  
          ```json
          {
            "categoryUrls": [
              {
                "url": "https://www.amazon.in/s?k={{ $json.amazonSearchPart }}",
                "method": "GET"
              }
            ],
            "ensureLoadedProductDescriptionFields": false,
            "maxItemsPerStartUrl": 10,
            "scrapeProductDetails": true,
            "scrapeProductVariantPrices": false,
            "useCaptchaSolver": false
          }
          ```
        - Replace `{{ $json.amazonSearchPart }}` dynamically from cleaner node.

   8.2. **Create Product Data Processor node (Code):**
        - Connect Amazon Product Scraper → Product Data Processor.  
        - JavaScript code to extract product title, price, stars, total reviews, URL, description.  
        - Calculate total reviews from stars breakdown percentages.

   8.3. **Create Product List Combiner node (Aggregate):**
        - Connect Product Data Processor → Product List Combiner.  
        - Aggregate all products into single array.

   8.4. **Create Google Gemini Chat Model3 node:**
        - Type: AI Language Model (Google Gemini)  
        - Connect Product List Combiner → Product Recommendation Engine (ai_languageModel).  
        - Use Google Palm API credentials.

   8.5. **Create Product Recommendation Engine node (AI Agent):**
        - Connect Product List Combiner → Product Recommendation Engine (main).  
        - Connect Google Gemini Chat Model3 → Product Recommendation Engine (ai_languageModel).  
        - Configure detailed system prompt to analyze products and user query, producing top 5 Telegram-ready recommendations with strict formatting.

   8.6. **Create Send Product Recommendations node:**
        - Type: Telegram node.  
        - Connect Product Recommendation Engine → Send Product Recommendations.  
        - Use Telegram API credentials.  
        - Set `chatId` to incoming message chat ID.  
        - Send AI recommendation text.

9. **Test workflow end-to-end with various Telegram messages:**
   - Product queries (e.g., "Bluetooth earphones on Amazon").  
   - Casual chat (e.g., "Hello", "Who are you?").  
   - Edge cases (empty messages, invalid queries).

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          | Context or Link                                                  |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------|
| 🛒 Intelligent Telegram Shopping Assistant Bot: The workflow includes multiple well-organized logical sections such as Input Reception, Query Cleanup, Intent Classification, Routing, Chat Response, Product Search, Data Processing, Aggregation, Recommendation, and Output Delivery, all powered by Google Gemini AI models. This modular design enables scalability and clear maintenance.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       | Summary of workflow logical sections                            |
| Sticky Note on AI Models Integration: Utilizes three specialized Google Gemini AI models for chat responses, intent detection, and product analysis, ensuring a consistent and powerful AI experience.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                | Integration details                                             |
| Project uses Apify's Free Amazon Product Scraper API for real-time product data extraction, enabling up to 10 products per search query without requiring own scraping infrastructure.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  | Apify Amazon Scraper API                                         |
| Telegram API integration requires valid bot token credentials with proper webhook configuration.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  | Telegram API credentials and webhook setup                      |
| AI prompt engineering enforces strict JSON outputs for intent classification and limits message lengths and formats for product recommendations to fit Telegram constraints.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       | AI prompt guidelines and constraints                            |
| Error handling is partially addressed with validation in the query cleaner and structured parsing, but additional monitoring for API failures and empty results is recommended.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       | Recommendations for error handling                              |
| Branding and user instructions are embedded in AI system messages guiding the assistant to maintain focus on shopping assistance and avoid unnecessary chit-chat.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            | AI behavior and response design notes                           |

---

**Disclaimer:** The provided text is exclusively derived from an automated n8n workflow. It complies strictly with current content policies and contains no illegal, offensive, or protected elements. All data processed is legal and public.