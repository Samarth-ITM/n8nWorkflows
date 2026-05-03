Compare product prices from Amazon, Walmart and Google Shopping via Telegram using OpenAI

https://n8nworkflows.xyz/workflows/compare-product-prices-from-amazon--walmart-and-google-shopping-via-telegram-using-openai-14330


# Compare product prices from Amazon, Walmart and Google Shopping via Telegram using OpenAI

# 1. Workflow Overview

This workflow turns a Telegram bot into a product price comparison assistant. A user sends a product name in Telegram, the workflow searches Amazon, Walmart, and Google Shopping through SerpApi, normalizes the marketplace results into a common format, asks an OpenAI-powered agent to determine the best sensible offer, then replies in Telegram with a recommendation.

Typical use cases:
- Comparing consumer product prices across major marketplaces
- Filtering weak or suspicious offers before recommending a purchase
- Returning a concise, human-readable buying recommendation inside Telegram

## 1.1 Input Reception

The workflow starts when a Telegram user sends a message to the bot. That message text is treated as the product search query and is passed to all marketplace search branches in parallel.

## 1.2 Marketplace Search and Normalization

Three HTTP Request nodes query SerpApi separately for:
- Google Shopping
- Amazon
- Walmart

Each marketplace response is then parsed by a Code node that converts results into a shared offer schema. This is essential because each SerpApi engine returns different field names and structures.

## 1.3 Offer Consolidation

The normalized outputs from all three marketplace branches are merged and aggregated into one dataset. This creates a single input structure for AI evaluation.

## 1.4 AI Offer Evaluation

A LangChain Agent node uses an OpenAI chat model to review all collected offers. The prompt instructs the model to:
- Compare only valid matching variants
- Avoid suspiciously cheap or weak listings
- Return strict JSON with pricing statistics and a recommendation

## 1.5 Output Formatting and Delivery

The AI output is parsed and reformatted into a Telegram-friendly message. If the AI output is invalid or no strong match is found, the workflow falls back to a helpful clarification message. The final response is sent back to the same Telegram chat.

---

# 2. Block-by-Block Analysis

## 2.1 Input Reception

### Overview
This block receives incoming Telegram messages and uses the message text as the product query. It is the workflow’s only entry point.

### Nodes Involved
- Telegram Product Request

### Node Details

#### Telegram Product Request
- **Type and technical role:** `n8n-nodes-base.telegramTrigger`  
  Trigger node that starts the workflow when the Telegram bot receives a message.
- **Configuration choices:**
  - Trigger listens for `message` updates
  - No extra filtering is configured
- **Key expressions or variables used:**
  - Downstream nodes use `{{$json.message.text}}` as the product name
  - Chat ID later comes from `$('Telegram Product Request').item.json.message.chat.id`
- **Input and output connections:**
  - Input: none, trigger node
  - Outputs to:
    - Search Walmart via SerpApi
    - Search Amazon via SerpApi
    - Search Google Shopping via SerpApi
- **Version-specific requirements:**
  - Uses node type version `1.2`
  - Requires valid Telegram bot credentials configured in n8n
  - Requires webhook registration to Telegram
- **Edge cases or potential failure types:**
  - Missing or invalid Telegram credentials
  - Bot webhook issues
  - User sends a non-text message; downstream expressions relying on `message.text` may fail or produce empty queries
  - Telegram bot privacy settings may prevent the bot from receiving expected messages in groups
- **Sub-workflow reference:** none

---

## 2.2 Marketplace Search and Normalization

### Overview
This block performs three parallel searches through SerpApi and converts each marketplace’s response into a consistent structure. The normalization step is critical because the AI block expects comparable offer objects regardless of original source.

### Nodes Involved
- Search Google Shopping via SerpApi
- Parse Google Shopping Response
- Search Amazon via SerpApi
- Parse Amazon Response
- Search Walmart via SerpApi
- Parse Walmart Response

### Node Details

#### Search Google Shopping via SerpApi
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Calls SerpApi’s Google Shopping engine.
- **Configuration choices:**
  - Method defaults to GET through query parameters
  - URL: `https://serpapi.com/search`
  - Response format set to JSON
  - Query parameters:
    - `engine=google_shopping`
    - `q={{ $json.message.text }}`
    - `location=United+States`
    - `api_key` must be filled manually
- **Key expressions or variables used:**
  - `{{$json.message.text}}` from Telegram trigger
- **Input and output connections:**
  - Input: Telegram Product Request
  - Output: Parse Google Shopping Response
- **Version-specific requirements:**
  - Uses HTTP Request node version `4.4`
- **Edge cases or potential failure types:**
  - Missing SerpApi key
  - Invalid location value for target market
  - Empty query if Telegram message text is absent
  - Rate limiting or quota exhaustion on SerpApi
  - Google Shopping results may appear under different keys depending on query/result type
- **Sub-workflow reference:** none

#### Parse Google Shopping Response
- **Type and technical role:** `n8n-nodes-base.code`  
  Extracts shopping results from possible SerpApi response fields and maps them into a standard offer format.
- **Configuration choices:**
  - Tries multiple candidate arrays:
    - `shopping_results`
    - `inline_shopping_results`
    - `products`
    - fallback to `[]`
  - Keeps only entries with both `title` and `extracted_price`
  - Maps to:
    - `marketplace`
    - `product_id`
    - `offer_id`
    - `title`
    - `offer_price`
    - `reviews`
    - `seller_name`
    - `badge`
    - `product_url`
- **Key expressions or variables used:**
  - `$input.first().json...`
- **Input and output connections:**
  - Input: Search Google Shopping via SerpApi
  - Output: Merge Marketplace Offers
- **Version-specific requirements:**
  - Code node version `2`
- **Edge cases or potential failure types:**
  - Empty result arrays
  - Changed SerpApi response shape
  - Missing `extracted_price` causing valid listings to be excluded
  - `source` may represent a store rather than marketplace, which can affect AI interpretation
- **Sub-workflow reference:** none

#### Search Amazon via SerpApi
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Calls SerpApi’s Amazon engine.
- **Configuration choices:**
  - URL: `https://serpapi.com/search`
  - JSON response enabled
  - Query parameters:
    - `engine=amazon`
    - `k={{ $json.message.text }}`
    - `amazon_domain=amazon.com`
    - `api_key` must be filled manually
- **Key expressions or variables used:**
  - `{{$json.message.text}}`
- **Input and output connections:**
  - Input: Telegram Product Request
  - Output: Parse Amazon Response
- **Version-specific requirements:**
  - HTTP Request version `4.4`
- **Edge cases or potential failure types:**
  - Missing API key
  - SerpApi Amazon engine quota/rate limit
  - Region/domain mismatch for intended audience
  - Empty or malformed query
- **Sub-workflow reference:** none

#### Parse Amazon Response
- **Type and technical role:** `n8n-nodes-base.code`  
  Filters and normalizes Amazon organic results.
- **Configuration choices:**
  - Reads `organic_results`
  - Filters out items without a `price`
  - Excludes titles containing blocked words:
    - `Clear`
    - `Case`
  - Maps to unified fields:
    - `marketplace: "Amazon"`
    - `product_id` and `offer_id` from `asin`
    - `offer_price` from `extracted_price`
    - `seller_name: "Amazon"`
    - `badge: "Overall Pick"`
    - URL from `link_clean` or `link`
- **Key expressions or variables used:**
  - `$input.first().json.organic_results`
- **Input and output connections:**
  - Input: Search Amazon via SerpApi
  - Output: Merge Marketplace Offers
- **Version-specific requirements:**
  - Code node version `2`
- **Edge cases or potential failure types:**
  - Case-sensitive filtering bug: blocked array contains `Clear` and `Case`, but title is lowercased before checking; this means these terms will not actually match as written
  - Missing `organic_results`
  - Amazon result titles may include accessories or alternate variants that survive the simple filter
  - Seller is hardcoded to `Amazon`, even if actual seller differs
  - Badge is hardcoded to `Overall Pick`, which may overstate metadata quality
- **Sub-workflow reference:** none

#### Search Walmart via SerpApi
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Calls SerpApi’s Walmart engine.
- **Configuration choices:**
  - URL: `https://serpapi.com/search`
  - JSON response enabled
  - Query parameters:
    - `engine=walmart`
    - `query={{ $json.message.text }}`
    - `api_key` must be filled manually
- **Key expressions or variables used:**
  - `{{$json.message.text}}`
- **Input and output connections:**
  - Input: Telegram Product Request
  - Output: Parse Walmart Response
- **Version-specific requirements:**
  - HTTP Request version `4.4`
- **Edge cases or potential failure types:**
  - Missing API key
  - Quota/rate-limit errors
  - Walmart engine data availability varies by product category
- **Sub-workflow reference:** none

#### Parse Walmart Response
- **Type and technical role:** `n8n-nodes-base.code`  
  Filters and normalizes Walmart organic results.
- **Configuration choices:**
  - Reads `organic_results`
  - Keeps only items where `seller_name` exists and is not empty
  - Excludes titles containing blocked carrier names:
    - `at&t`
    - `at&amp;t`
    - `verizon`
    - `t-mobile`
  - Maps fields into the common structure:
    - `seller_name`
    - `product_id`
    - `offer_id`
    - `title`
    - `offer_price`
    - `reviews`
    - `badge: null`
    - `product_url`
- **Key expressions or variables used:**
  - `$input.first().json.organic_results`
- **Input and output connections:**
  - Input: Search Walmart via SerpApi
  - Output: Merge Marketplace Offers
- **Version-specific requirements:**
  - Code node version `2`
- **Edge cases or potential failure types:**
  - If `organic_results` is missing, code may error because `.filter` would be called on `undefined`
  - The node does not explicitly assign a `marketplace` field, unlike Amazon and Google Shopping mappings
  - Carrier filtering is useful for phone searches but may accidentally exclude legitimate locked-device use cases
  - Some valid Walmart listings may be removed if `seller_name` is absent
- **Sub-workflow reference:** none

---

## 2.3 Offer Consolidation

### Overview
This block combines the normalized offers from all marketplace branches and aggregates them into a single object. The AI node consumes this unified dataset rather than multiple item streams.

### Nodes Involved
- Merge Marketplace Offers
- Aggregate Offer Data

### Node Details

#### Merge Marketplace Offers
- **Type and technical role:** `n8n-nodes-base.merge`  
  Combines three incoming item streams into one output stream.
- **Configuration choices:**
  - `numberInputs: 3`
  - Inputs are connected as:
    - Input 0: Parse Walmart Response
    - Input 1: Parse Amazon Response
    - Input 2: Parse Google Shopping Response
- **Key expressions or variables used:** none
- **Input and output connections:**
  - Inputs:
    - Parse Walmart Response
    - Parse Amazon Response
    - Parse Google Shopping Response
  - Output:
    - Aggregate Offer Data
- **Version-specific requirements:**
  - Merge node version `3.2`
- **Edge cases or potential failure types:**
  - If one branch returns no items, overall merge behavior depends on execution data arriving from all branches; in this workflow it should still proceed once upstream paths resolve, but practical output volume may vary
  - Data inconsistency between branches can affect downstream AI quality
- **Sub-workflow reference:** none

#### Aggregate Offer Data
- **Type and technical role:** `n8n-nodes-base.aggregate`  
  Converts all merged items into one item containing an array of offer objects.
- **Configuration choices:**
  - Aggregate mode: `aggregateAllItemData`
  - No special options set
- **Key expressions or variables used:**
  - Downstream AI prompt refers to `{{$json.data}}`
- **Input and output connections:**
  - Input: Merge Marketplace Offers
  - Output: AI: Choose Best Offer
- **Version-specific requirements:**
  - Aggregate node version `1`
- **Edge cases or potential failure types:**
  - If zero items reach the node, the resulting structure may still exist but contain an empty array
  - Large result sets increase token usage in the AI step
- **Sub-workflow reference:** none

---

## 2.4 AI Offer Evaluation

### Overview
This block uses an OpenAI chat model through the n8n LangChain agent node to analyze all collected offers and choose the best sensible recommendation. The prompt is designed to avoid naïvely selecting the absolute lowest price when the match looks risky or incorrect.

### Nodes Involved
- Price Analysis Model
- AI: Choose Best Offer

### Node Details

#### Price Analysis Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
  Provides the language model used by the agent.
- **Configuration choices:**
  - Model selected: `gpt-5-mini`
  - No additional options or built-in tools configured
- **Key expressions or variables used:** none
- **Input and output connections:**
  - Connected through AI language model port into AI: Choose Best Offer
- **Version-specific requirements:**
  - LangChain OpenAI Chat node version `1.3`
  - Requires OpenAI credentials configured in n8n
  - Availability of `gpt-5-mini` depends on account/model access and current n8n/OpenAI compatibility
- **Edge cases or potential failure types:**
  - Invalid or missing OpenAI credentials
  - Model unavailable in account/region
  - Token limits if marketplace data grows too large
  - Cost considerations for frequent usage
- **Sub-workflow reference:** none

#### AI: Choose Best Offer
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`  
  Uses the model to evaluate all offers and produce a strict JSON recommendation.
- **Configuration choices:**
  - Prompt type: defined text
  - Injects the full aggregated array using:
    - `{{ JSON.stringify($json.data) }}`
  - The prompt instructs the model to:
    - Use only provided input
    - Compare title, price, seller, reviews, URL
    - Exclude non-matching variants
    - Mark condition from title where possible
    - Choose the lowest sensible price
    - Return valid JSON only
  - Expected JSON schema includes:
    - `product_name`
    - `total_offers_reviewed`
    - `comparable_offer_count`
    - `min_price`
    - `max_price`
    - `avg_price`
    - `median_price`
    - `recommended_price`
    - `recommended_seller`
    - `recommended_url`
    - `condition_summary`
    - `confidence`
    - `reasoning`
- **Key expressions or variables used:**
  - `{{ JSON.stringify($json.data) }}`
  - Output later consumed from `json.output`
- **Input and output connections:**
  - Main input: Aggregate Offer Data
  - AI model input: Price Analysis Model
  - Main output: Format AI Recommendation
- **Version-specific requirements:**
  - Agent node version `3.1`
  - Requires compatible LangChain package support in the n8n version used
- **Edge cases or potential failure types:**
  - LLM may return invalid JSON despite prompt constraints
  - The model may over- or under-filter offers based on ambiguous titles
  - Lack of explicit marketplace field on Walmart items can reduce reasoning quality
  - If aggregated data is empty, model may still fabricate structure unless the prompt is carefully obeyed
- **Sub-workflow reference:** none

---

## 2.5 Output Formatting and Delivery

### Overview
This block converts the AI JSON output into a user-friendly Telegram message and sends it back to the original chat. It also includes fallback handling for invalid AI responses or weak product matches.

### Nodes Involved
- Format AI Recommendation
- Send Telegram Recommendation

### Node Details

#### Format AI Recommendation
- **Type and technical role:** `n8n-nodes-base.code`  
  Parses the AI response and formats a Telegram message.
- **Configuration choices:**
  - Reads raw AI output from `$input.first().json.output`
  - Attempts `JSON.parse(raw)`
  - On parse failure, returns:
    - “I couldn't parse the recommendation result.”
    - Prompt to retry with a more specific product name
  - Formats numeric prices with `$` and two decimals
  - Builds fields:
    - confidence
    - reasoning
    - comparable count
    - product name
    - recommended price
    - min/max price
    - seller
    - link
  - If no recommendation and no comparable offers exist, returns a “no confident product match” message
  - Otherwise constructs a final multi-line response
- **Key expressions or variables used:**
  - `$input.first().json.output`
- **Input and output connections:**
  - Input: AI: Choose Best Offer
  - Output: Send Telegram Recommendation
- **Version-specific requirements:**
  - Code node version `2`
- **Edge cases or potential failure types:**
  - AI output not in `output` field in some agent configurations
  - AI returns valid JSON but with wrong field names
  - `0` prices would be treated as falsy in the no-match condition, though not realistic here
  - Malformed URLs are passed through directly
- **Sub-workflow reference:** none

#### Send Telegram Recommendation
- **Type and technical role:** `n8n-nodes-base.telegram`  
  Sends the final recommendation message back to Telegram.
- **Configuration choices:**
  - Text: `={{ $json.message }}`
  - Chat ID: `={{ $('Telegram Product Request').item.json.message.chat.id }}`
  - No additional fields configured
- **Key expressions or variables used:**
  - Message body from current item
  - Chat ID pulled directly from trigger node execution data
- **Input and output connections:**
  - Input: Format AI Recommendation
  - Output: none
- **Version-specific requirements:**
  - Telegram node version `1.2`
  - Requires same or compatible Telegram credentials as trigger
- **Edge cases or potential failure types:**
  - Bot blocked by user
  - Invalid chat reference if execution context changes unexpectedly
  - Telegram message length limits if reasoning becomes too long
- **Sub-workflow reference:** none

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note | n8n-nodes-base.stickyNote | Workspace documentation / overview note |  |  | ## 🛒 Find the Best Product Price Across Amazon, Walmart & Google Shopping<br><br>Send a product name to Telegram, compare offers across Amazon, Walmart, and Google Shopping, and get the best sensible buying recommendation with confidence level, price range, and a direct purchase link.<br><br>**What this template does**<br>This template turns Telegram into a smart product price comparison tool. When a user sends a product name, the workflow simultaneously queries Amazon, Walmart, and Google Shopping, standardizes all results into a unified format, and delivers the most practical buying recommendation — filtering out suspicious listings and weak matches using an AI step.<br><br>**How it works**<br>1. A user sends a product name to the Telegram bot<br>2. The workflow queries Walmart, Amazon, and Google Shopping in parallel<br>3. Each marketplace response is cleaned and mapped into a consistent offer structure<br>4. All offers are merged and consolidated into a single dataset<br>5. An AI agent evaluates price, seller, reviews, and match quality to identify the best sensible offer<br>6. A Telegram message is sent back with the recommended price, seller, price range, offer count, confidence level, reasoning, and a direct purchase link<br><br>**Requirements:** Telegram API · OpenAI · SerpApi — add your SerpApi key and review any fixed location/region values in the HTTP Request nodes before use.<br><br>**Good to know:** More specific product names yield more accurate results. If no strong match is found, the workflow prompts the user to refine their query. |
| Sticky Note1 | n8n-nodes-base.stickyNote | Workspace documentation for input block |  |  | ## 📨 Input<br><br>User sends a product name via Telegram to trigger the workflow. |
| Sticky Note2 | n8n-nodes-base.stickyNote | Workspace documentation for marketplace search block |  |  | ## 🔍 Marketplace Search<br><br>Searches Google Shopping, Amazon, and Walmart in parallel via SerpApi.<br><br>Each response is cleaned and normalized into a shared offer structure for consistent comparison.<br><br>⚙️ Add your **SerpApi** key to each HTTP Request node and adjust any fixed location or region values as needed. |
| Sticky Note3 | n8n-nodes-base.stickyNote | Workspace documentation for AI block |  |  | ## 🤖 AI Offer Comparison<br><br>Merges all marketplace results, then uses an AI agent (GPT-4) to evaluate price, seller reputation, reviews, and title match quality.<br><br>Avoids suspiciously cheap listings, weak matches, and clearly different variants — recommending the best sensible buying option. |
| Sticky Note4 | n8n-nodes-base.stickyNote | Workspace documentation for output block |  |  | ## 📤 Output<br><br>Formats the AI recommendation and sends a Telegram message with: best price, seller, price range, offer count, confidence level, reasoning, and purchase link. |
| Telegram Product Request | n8n-nodes-base.telegramTrigger | Entry trigger from Telegram user message |  | Search Walmart via SerpApi; Search Amazon via SerpApi; Search Google Shopping via SerpApi | ## 📨 Input<br><br>User sends a product name via Telegram to trigger the workflow. |
| Search Walmart via SerpApi | n8n-nodes-base.httpRequest | Search Walmart product listings through SerpApi | Telegram Product Request | Parse Walmart Response | ## 🔍 Marketplace Search<br><br>Searches Google Shopping, Amazon, and Walmart in parallel via SerpApi.<br><br>Each response is cleaned and normalized into a shared offer structure for consistent comparison.<br><br>⚙️ Add your **SerpApi** key to each HTTP Request node and adjust any fixed location or region values as needed. |
| Parse Walmart Response | n8n-nodes-base.code | Normalize Walmart search results into shared offer schema | Search Walmart via SerpApi | Merge Marketplace Offers | ## 🔍 Marketplace Search<br><br>Searches Google Shopping, Amazon, and Walmart in parallel via SerpApi.<br><br>Each response is cleaned and normalized into a shared offer structure for consistent comparison.<br><br>⚙️ Add your **SerpApi** key to each HTTP Request node and adjust any fixed location or region values as needed. |
| Search Amazon via SerpApi | n8n-nodes-base.httpRequest | Search Amazon product listings through SerpApi | Telegram Product Request | Parse Amazon Response | ## 🔍 Marketplace Search<br><br>Searches Google Shopping, Amazon, and Walmart in parallel via SerpApi.<br><br>Each response is cleaned and normalized into a shared offer structure for consistent comparison.<br><br>⚙️ Add your **SerpApi** key to each HTTP Request node and adjust any fixed location or region values as needed. |
| Parse Amazon Response | n8n-nodes-base.code | Normalize Amazon search results into shared offer schema | Search Amazon via SerpApi | Merge Marketplace Offers | ## 🔍 Marketplace Search<br><br>Searches Google Shopping, Amazon, and Walmart in parallel via SerpApi.<br><br>Each response is cleaned and normalized into a shared offer structure for consistent comparison.<br><br>⚙️ Add your **SerpApi** key to each HTTP Request node and adjust any fixed location or region values as needed. |
| Search Google Shopping via SerpApi | n8n-nodes-base.httpRequest | Search Google Shopping listings through SerpApi | Telegram Product Request | Parse Google Shopping Response | ## 🔍 Marketplace Search<br><br>Searches Google Shopping, Amazon, and Walmart in parallel via SerpApi.<br><br>Each response is cleaned and normalized into a shared offer structure for consistent comparison.<br><br>⚙️ Add your **SerpApi** key to each HTTP Request node and adjust any fixed location or region values as needed. |
| Parse Google Shopping Response | n8n-nodes-base.code | Normalize Google Shopping search results into shared offer schema | Search Google Shopping via SerpApi | Merge Marketplace Offers | ## 🔍 Marketplace Search<br><br>Searches Google Shopping, Amazon, and Walmart in parallel via SerpApi.<br><br>Each response is cleaned and normalized into a shared offer structure for consistent comparison.<br><br>⚙️ Add your **SerpApi** key to each HTTP Request node and adjust any fixed location or region values as needed. |
| Merge Marketplace Offers | n8n-nodes-base.merge | Combine normalized offers from all marketplace branches | Parse Walmart Response; Parse Amazon Response; Parse Google Shopping Response | Aggregate Offer Data | ## 🤖 AI Offer Comparison<br><br>Merges all marketplace results, then uses an AI agent (GPT-4) to evaluate price, seller reputation, reviews, and title match quality.<br><br>Avoids suspiciously cheap listings, weak matches, and clearly different variants — recommending the best sensible buying option. |
| Aggregate Offer Data | n8n-nodes-base.aggregate | Aggregate all offers into one array for AI analysis | Merge Marketplace Offers | AI: Choose Best Offer | ## 🤖 AI Offer Comparison<br><br>Merges all marketplace results, then uses an AI agent (GPT-4) to evaluate price, seller reputation, reviews, and title match quality.<br><br>Avoids suspiciously cheap listings, weak matches, and clearly different variants — recommending the best sensible buying option. |
| Price Analysis Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | OpenAI chat model backing the AI agent |  | AI: Choose Best Offer | ## 🤖 AI Offer Comparison<br><br>Merges all marketplace results, then uses an AI agent (GPT-4) to evaluate price, seller reputation, reviews, and title match quality.<br><br>Avoids suspiciously cheap listings, weak matches, and clearly different variants — recommending the best sensible buying option. |
| AI: Choose Best Offer | @n8n/n8n-nodes-langchain.agent | Evaluate offers and produce structured recommendation JSON | Aggregate Offer Data; Price Analysis Model | Format AI Recommendation | ## 🤖 AI Offer Comparison<br><br>Merges all marketplace results, then uses an AI agent (GPT-4) to evaluate price, seller reputation, reviews, and title match quality.<br><br>Avoids suspiciously cheap listings, weak matches, and clearly different variants — recommending the best sensible buying option. |
| Format AI Recommendation | n8n-nodes-base.code | Parse AI output and build Telegram-friendly message | AI: Choose Best Offer | Send Telegram Recommendation | ## 📤 Output<br><br>Formats the AI recommendation and sends a Telegram message with: best price, seller, price range, offer count, confidence level, reasoning, and purchase link. |
| Send Telegram Recommendation | n8n-nodes-base.telegram | Send final recommendation back to Telegram chat | Format AI Recommendation |  | ## 📤 Output<br><br>Formats the AI recommendation and sends a Telegram message with: best price, seller, price range, offer count, confidence level, reasoning, and purchase link. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it something like `Best Price`.
   - Optional: add sticky notes for documentation and visual grouping.

2. **Add the Telegram trigger**
   - Create a **Telegram Trigger** node named `Telegram Product Request`.
   - Configure Telegram credentials using your bot token.
   - Set the trigger to listen for `message` updates.
   - Keep other settings at default unless you need special filtering.
   - Test the node by sending a message to your bot.

3. **Create the Google Shopping search branch**
   - Add an **HTTP Request** node named `Search Google Shopping via SerpApi`.
   - Connect it from `Telegram Product Request`.
   - Set:
     - URL: `https://serpapi.com/search`
     - Response format: JSON
     - Send Query Parameters: enabled
   - Add query parameters:
     - `engine` = `google_shopping`
     - `q` = `={{ $json.message.text }}`
     - `location` = `United+States`
     - `api_key` = your SerpApi key
   - If your use case is not US-focused, change `location`.

4. **Add the Google Shopping parsing node**
   - Add a **Code** node named `Parse Google Shopping Response`.
   - Connect it from `Search Google Shopping via SerpApi`.
   - Paste logic that:
     - Reads from `shopping_results`, `inline_shopping_results`, or `products`
     - Filters out items missing title or extracted price
     - Maps each item to a unified offer object with:
       - `marketplace`
       - `product_id`
       - `offer_id`
       - `title`
       - `offer_price`
       - `reviews`
       - `seller_name`
       - `badge`
       - `product_url`

5. **Create the Amazon search branch**
   - Add an **HTTP Request** node named `Search Amazon via SerpApi`.
   - Connect it from `Telegram Product Request`.
   - Set:
     - URL: `https://serpapi.com/search`
     - Response format: JSON
     - Send Query Parameters: enabled
   - Add query parameters:
     - `engine` = `amazon`
     - `k` = `={{ $json.message.text }}`
     - `amazon_domain` = `amazon.com`
     - `api_key` = your SerpApi key

6. **Add the Amazon parsing node**
   - Add a **Code** node named `Parse Amazon Response`.
   - Connect it from `Search Amazon via SerpApi`.
   - Implement logic that:
     - Reads `organic_results`
     - Filters out results without price
     - Excludes obvious accessory keywords like case/clear if desired
     - Maps output fields to the same shared schema used in the Google branch
   - Important improvement recommended:
     - If you lowercase titles before matching, lowercase the blocked keywords too, otherwise filtering will not work as intended.

7. **Create the Walmart search branch**
   - Add an **HTTP Request** node named `Search Walmart via SerpApi`.
   - Connect it from `Telegram Product Request`.
   - Set:
     - URL: `https://serpapi.com/search`
     - Response format: JSON
     - Send Query Parameters: enabled
   - Add query parameters:
     - `engine` = `walmart`
     - `query` = `={{ $json.message.text }}`
     - `api_key` = your SerpApi key

8. **Add the Walmart parsing node**
   - Add a **Code** node named `Parse Walmart Response`.
   - Connect it from `Search Walmart via SerpApi`.
   - Implement logic that:
     - Reads `organic_results`
     - Excludes items without `seller_name`
     - Excludes blocked carrier terms if you want to avoid phone-contract results
     - Maps data into the same shared schema
   - Recommended improvement:
     - Add `marketplace: "Walmart"` explicitly for consistency.
     - Safeguard missing arrays by using `const results = $input.first().json.organic_results || [];`

9. **Merge marketplace outputs**
   - Add a **Merge** node named `Merge Marketplace Offers`.
   - Set number of inputs to `3`.
   - Connect:
     - `Parse Walmart Response` to input 1
     - `Parse Amazon Response` to input 2
     - `Parse Google Shopping Response` to input 3
   - The goal is to collect all normalized offers into one stream.

10. **Aggregate all offer items**
    - Add an **Aggregate** node named `Aggregate Offer Data`.
    - Connect it from `Merge Marketplace Offers`.
    - Set aggregation mode to **Aggregate All Item Data**.
    - This should produce one item containing a `data` array of all offers.

11. **Add the OpenAI model node**
    - Add an **OpenAI Chat Model** node from the LangChain category named `Price Analysis Model`.
    - Configure OpenAI credentials.
    - Select model `gpt-5-mini` or another supported chat model available in your account.
    - Leave advanced options default unless you need stricter controls.

12. **Add the AI agent node**
    - Add an **AI Agent** node named `AI: Choose Best Offer`.
    - Connect:
      - Main input from `Aggregate Offer Data`
      - AI language model input from `Price Analysis Model`
    - Set prompt type to a custom defined prompt.
    - Paste a prompt that:
      - Describes the role as product price analyst
      - Injects `{{ JSON.stringify($json.data) }}`
      - Requires comparison by title, price, seller, reviews, and URL
      - Excludes mismatched variants and unclear prices
      - Returns strict JSON only
    - Use the same expected JSON schema as in the source workflow:
      - `product_name`
      - `total_offers_reviewed`
      - `comparable_offer_count`
      - `min_price`
      - `max_price`
      - `avg_price`
      - `median_price`
      - `recommended_price`
      - `recommended_seller`
      - `recommended_url`
      - `condition_summary`
      - `confidence`
      - `reasoning`

13. **Add the response formatting code**
    - Add a **Code** node named `Format AI Recommendation`.
    - Connect it from `AI: Choose Best Offer`.
    - Implement logic that:
      - Reads AI output from the agent output field, typically `json.output`
      - Attempts to parse JSON
      - On parse failure, returns a user-friendly retry message
      - Formats prices as dollar values
      - Detects “no confident match” scenarios
      - Builds the final multi-line Telegram text

14. **Add the Telegram send node**
    - Add a **Telegram** node named `Send Telegram Recommendation`.
    - Connect it from `Format AI Recommendation`.
    - Reuse or configure Telegram credentials.
    - Set:
      - Text = `={{ $json.message }}`
      - Chat ID = `={{ $('Telegram Product Request').item.json.message.chat.id }}`
    - Leave additional fields empty unless you want formatting options.

15. **Credential setup**
    - Ensure these credentials are available:
      - **Telegram Bot API** for both trigger and send nodes
      - **OpenAI API** for the chat model node
      - **SerpApi key** entered in each HTTP Request node as query parameter `api_key`
    - For maintainability, consider moving SerpApi key usage to a credential or environment variable pattern if your n8n setup supports it.

16. **Test with real product messages**
    - Send sample inputs through Telegram such as:
      - `Sony WH-1000XM5`
      - `iPhone 15 128GB black unlocked`
      - `Samsung 990 Pro 2TB`
    - Confirm:
      - All three HTTP requests return data
      - Code nodes output normalized offers
      - Aggregate node produces a single `data` array
      - AI node returns valid JSON
      - Telegram node sends a readable message

17. **Recommended hardening before production**
    - Add an IF node before marketplace requests to ensure `message.text` exists.
    - Add error handling around HTTP request failures.
    - Add timeouts/retries for SerpApi requests.
    - Normalize marketplace field consistently across all parsers.
    - Fix Amazon blocked keyword matching by lowercasing the blocked list.
    - Add fallback behavior if one marketplace returns no results but others succeed.

### Sub-workflow setup
This workflow does **not** invoke any sub-workflows and does not require any child workflow configuration.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| The workflow is designed as a Telegram-based price comparison assistant across Amazon, Walmart, and Google Shopping. | General workflow purpose |
| Requirements explicitly mentioned in the workflow notes: Telegram API, OpenAI, SerpApi. | External services required |
| The SerpApi key must be inserted manually into each HTTP Request node in the current design. | Search nodes configuration |
| The Google Shopping branch includes a fixed `United+States` location value, and Amazon uses `amazon.com`; these should be adjusted for other regions. | Marketplace localization |
| More specific product names yield better comparisons and more reliable AI recommendations. | Usage guidance |
| If no strong match is found, the workflow returns a refinement prompt asking for more precise brand/model/variant details. | End-user behavior |
| The sticky note says “AI agent (GPT-4)”, but the actual configured model node is `gpt-5-mini`. | Important implementation detail |