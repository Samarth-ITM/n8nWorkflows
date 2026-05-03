Extract Amazon Product Data with Scrape.do, GPT-4 & Google Sheets

https://n8nworkflows.xyz/workflows/extract-amazon-product-data-with-scrape-do--gpt-4---google-sheets-8678


# Extract Amazon Product Data with Scrape.do, GPT-4 & Google Sheets

---

### 1. Workflow Overview

This workflow automates the extraction of structured product data from Amazon product pages listed in a Google Sheet. It leverages the Scrape.do API to retrieve raw HTML content bypassing Amazon’s anti-bot restrictions, then uses HTML extraction and GPT-4 based AI processing to clean and structure the product information. Finally, the structured data is appended back into another tab of the same Google Sheet.

**Target Use Cases:**
- Automating Amazon product data collection for market research, price monitoring, or inventory management.
- Bypassing Amazon anti-scraping mechanisms without managing proxies manually.
- Structuring semi-structured product data using AI for downstream analysis.

**Logical Blocks:**

- **1. Input Reception & Initialization**
  - Triggering the workflow manually.
  - Fetching product URLs from a Google Sheet.

- **2. Iterative Processing**
  - Looping over each URL to process products sequentially in batches.

- **3. Data Retrieval & Extraction**
  - Calling Scrape.do API to get raw product page HTML.
  - Extracting relevant raw data elements using CSS selectors.

- **4. AI-based Data Cleaning & Structuring**
  - Using GPT-4 (via Langchain n8n nodes) to clean, parse, and format product data into a consistent JSON schema.

- **5. Data Output & Persistence**
  - Formatting the final JSON output.
  - Appending structured product data back into a Google Sheet.

- **6. Documentation & Setup Instructions**
  - Included sticky note describing setup steps, credentials, and workflow features.

---

### 2. Block-by-Block Analysis

#### 1. Input Reception & Initialization

- **Overview:**  
  Starts the workflow manually and retrieves the list of Amazon product URLs from a Google Sheet for processing.

- **Nodes Involved:**  
  - When clicking Test workflow  
  - 1. Get Product URLs from Google Sheets  
  - 2. Loop Through Each URL

- **Node Details:**

  1. **When clicking Test workflow**  
     - Type: Manual Trigger  
     - Role: Initiates the workflow on demand.  
     - Inputs: None  
     - Outputs: Triggers downstream nodes.  
     - Failures: None typical; manual start only.

  2. **1. Get Product URLs from Google Sheets**  
     - Type: Google Sheets node (read operation)  
     - Role: Reads the first sheet (gid=0) of the Google Sheet document containing Amazon product URLs.  
     - Configuration:  
       * Document ID: `19Allmozbygw-QogPeq2TH9m9D57FCn4MTu3zmJukg1A`  
       * Sheet Name: `gid=0` (Sheet1)  
     - Credentials: Google Sheets OAuth2  
     - Output: JSON array of rows with a column named `url`.  
     - Failures: Authentication errors, sheet not found, empty or malformed data.  
     - Edge Cases: Empty sheet, missing `url` column.

  3. **2. Loop Through Each URL**  
     - Type: SplitInBatches  
     - Role: Processes each product URL individually or in small batches to manage API limits and processing flow.  
     - Configuration: Default batch size (usually 1).  
     - Inputs: List of URLs from previous node.  
     - Outputs: Streams one URL at a time downstream.  
     - Failures: None typical, but large input lists can increase runtime.

---

#### 2. Data Retrieval & Extraction

- **Overview:**  
  For each URL, retrieves the raw Amazon product page HTML using Scrape.do API and extracts raw product data using CSS selectors.

- **Nodes Involved:**  
  - 3. Scrape Product Page HTML  
  - 4. Extract Raw Data Elements

- **Node Details:**

  1. **3. Scrape Product Page HTML**  
     - Type: HTTP Request  
     - Role: Calls Scrape.do API to fetch rendered Amazon product page HTML.  
     - Configuration:  
       * URL constructed dynamically:  
         `https://api.scrape.do/?token={{$vars.SCRAPEDO_TOKEN}}&url={{ encodeURIComponent($json.url) }}&geoCode=us&render=false`  
       * Timeout: 60 seconds  
     - Inputs: Current URL from loop  
     - Outputs: Raw HTML content in response body  
     - Failures: Network errors, invalid token, API limit exceeded, timeout, malformed URLs.  
     - Edge Cases: CAPTCHA triggered by Scrape.do, geo-restrictions, empty or invalid HTML.

  2. **4. Extract Raw Data Elements**  
     - Type: HTML Extractor  
     - Role: Parses raw HTML to extract product title, price, rating, review count, feature bullets, and description using CSS selectors.  
     - Configuration: CSS selectors targeting Amazon page elements such as:  
       * productTitle: `#productTitle, h1[data-automation-id="product-title"], .product-title`  
       * price: `.a-price .a-offscreen, .a-price-whole, .a-price-fraction, .priceToPay .a-price .a-offscreen`  
       * rating: `.a-icon-alt, [data-hook="average-star-rating"], .a-star-medium .a-icon-alt`  
       * reviewCount: `[data-hook="total-review-count"], .a-link-normal[href*="customerReviews"], #acrCustomerReviewText`  
       * featureBullets: `#feature-bullets ul, .a-unordered-list.a-nostyle.a-vertical.feature`  
       * productDescription: `#productDescription, [data-feature-name="productDescription"], .product-description`  
     - Inputs: HTML from previous node  
     - Outputs: JSON with extracted fields  
     - Failures: CSS selectors not matching due to page layout changes, empty fields, malformed HTML.

---

#### 3. AI-based Data Cleaning & Structuring

- **Overview:**  
  Uses GPT-4 to clean, normalize, and structure the extracted raw data into a defined JSON schema with product name, description, rating, reviews, and price.

- **Nodes Involved:**  
  - Structured Output Parser  
  - OpenAI Chat Model  
  - 5. Clean & Structure Data with AI

- **Node Details:**

  1. **Structured Output Parser**  
     - Type: Langchain Structured Output Parser  
     - Role: Defines expected JSON schema for AI output parsing.  
     - Configuration:  
       * JSON schema requiring at least `name` string  
       * Optional fields: description (string), rating (number or null), reviews (integer or null), price (string or null)  
     - Inputs: AI model raw output  
     - Outputs: Parsed JSON conforming to schema  
     - Failures: Schema mismatch, invalid JSON returned by AI.

  2. **OpenAI Chat Model**  
     - Type: Langchain OpenAI Chat Model node  
     - Role: Runs GPT-4o-mini model to process prompts and generate structured JSON responses.  
     - Configuration:  
       * Model: gpt-4o-mini (a GPT-4 variant)  
       * Max tokens: 500  
       * Temperature: 0 (deterministic output)  
       * Response format: JSON object  
     - Inputs: Prompt with extracted raw data, instructions to extract specific fields.  
     - Outputs: AI-generated JSON.  
     - Failures: API key or quota issues, timeouts, malformed responses.

  3. **5. Clean & Structure Data with AI**  
     - Type: Langchain Chain LLM node  
     - Role: Sends extracted data to GPT-4 with a defined prompt to produce cleaned, validated JSON output.  
     - Configuration:  
       * Input text: JSON string of extracted fields  
       * Prompt instructs AI to extract:  
         - name from productTitle  
         - description from featureBullets or productDescription (max 150 chars, default "No description")  
         - rating as number, or null  
         - reviews as integer, or null  
         - price formatted with $ if missing, or null  
       * Output parser linked to Structured Output Parser node  
     - Inputs: Raw data JSON from previous extraction node  
     - Outputs: Clean, structured JSON output  
     - Failures: AI hallucination, parsing errors if AI returns malformed data.

---

#### 4. Data Output & Persistence

- **Overview:**  
  Formats the AI output into discrete fields and appends the structured product data into a second tab of the Google Sheet.

- **Nodes Involved:**  
  - 6. Format Final JSON Output  
  - 7. Save Product Data to Google Sheets

- **Node Details:**

  1. **6. Format Final JSON Output**  
     - Type: SplitOut  
     - Role: Extracts specific fields (name, description, rating, reviews, price) from AI output for downstream processing.  
     - Configuration:  
       * Field to split out: `output` (the AI-generated JSON)  
       * Fields to include: output.name, output.description, output.rating, output.reviews, output.price  
     - Inputs: AI output JSON  
     - Outputs: Flattened fields for Google Sheets appending  
     - Failures: Missing fields, null values.

  2. **7. Save Product Data to Google Sheets**  
     - Type: Google Sheets node (append operation)  
     - Role: Appends processed product data as new rows into the results tab of the Google Sheet.  
     - Configuration:  
       * Document ID: same as input sheet (`19Allmozbygw-QogPeq2TH9m9D57FCn4MTu3zmJukg1A`)  
       * Sheet Name: `838351250` (Sheet2 / results tab)  
       * Operation: append rows  
       * Auto-mapping of input fields to columns  
     - Credentials: Google Sheets OAuth2  
     - Inputs: Flattened JSON fields  
     - Outputs: Confirmation of append operation  
     - Failures: Authentication, sheet not found, quota exceeded.

---

#### 5. Documentation & Setup Instructions

- **Overview:**  
  Provides setup notes, credentials instructions, and workflow description to guide users.

- **Nodes Involved:**  
  - Sticky Note1

- **Node Details:**

  1. **Sticky Note1**  
     - Type: Sticky Note  
     - Role: Contains detailed setup instructions and feature descriptions.  
     - Content Highlights:  
       * How to obtain Scrape.do API token  
       * Required environment variables (SCRAPEDO_TOKEN, sheet IDs)  
       * Google Sheets setup details (two tabs: URLs and results)  
       * Credential configurations (Google Sheets OAuth2, OpenRouter API for GPT-4)  
       * Notes on Scrape.do advantages over other scraping solutions  
     - Inputs/Outputs: None  
     - Failures: None

---

### 3. Summary Table

| Node Name                     | Node Type                             | Functional Role                          | Input Node(s)                  | Output Node(s)                      | Sticky Note                                                                                     |
|-------------------------------|-------------------------------------|----------------------------------------|-------------------------------|------------------------------------|------------------------------------------------------------------------------------------------|
| When clicking Test workflow    | Manual Trigger                      | Starts workflow manually                | -                             | 1. Get Product URLs from Google Sheets |                                                                                                |
| 1. Get Product URLs from Google Sheets | Google Sheets (read)               | Reads product URLs from Google Sheet   | When clicking Test workflow    | 2. Loop Through Each URL            |                                                                                                |
| 2. Loop Through Each URL       | SplitInBatches                     | Processes URLs in batches               | 1. Get Product URLs from Google Sheets | 3. Scrape Product Page HTML         |                                                                                                |
| 3. Scrape Product Page HTML    | HTTP Request                      | Calls Scrape.do API to get raw HTML    | 2. Loop Through Each URL       | 4. Extract Raw Data Elements        |                                                                                                |
| 4. Extract Raw Data Elements   | HTML Extractor                    | Extracts raw product data via CSS       | 3. Scrape Product Page HTML    | 5. Clean & Structure Data with AI   |                                                                                                |
| Structured Output Parser       | Langchain Output Parser           | Defines JSON schema for AI output      | OpenAI Chat Model              | 5. Clean & Structure Data with AI   |                                                                                                |
| OpenAI Chat Model              | Langchain LLM Chat Model          | Runs GPT-4 to clean and structure data | 5. Clean & Structure Data with AI (ai_languageModel input) | 5. Clean & Structure Data with AI (ai_outputParser input) |                                                                                                |
| 5. Clean & Structure Data with AI | Langchain Chain LLM               | Sends data to AI and parses output     | 4. Extract Raw Data Elements, Structured Output Parser, OpenAI Chat Model | 6. Format Final JSON Output          |                                                                                                |
| 6. Format Final JSON Output    | SplitOut                         | Flattens AI JSON output fields         | 5. Clean & Structure Data with AI | 7. Save Product Data to Google Sheets |                                                                                                |
| 7. Save Product Data to Google Sheets | Google Sheets (append)             | Saves structured data to Google Sheet  | 6. Format Final JSON Output    | 2. Loop Through Each URL            |                                                                                                |
| Sticky Note1                  | Sticky Note                      | Setup instructions and workflow notes  | -                             | -                                  | ## Amazon Scraper with Scrape.do API; Setup Instructions; Features; Advantages; https://scrape.do |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger Node**  
   - Name: `When clicking Test workflow`  
   - Purpose: Manually start the workflow.

2. **Add Google Sheets Node to Read URLs**  
   - Name: `1. Get Product URLs from Google Sheets`  
   - Operation: Read rows  
   - Document ID: Enter your Google Sheet document ID containing Amazon URLs  
   - Sheet Name: Set to the first tab (e.g., `gid=0` or sheet name)  
   - Credentials: Configure Google Sheets OAuth2 with access to the sheet  
   - Expected Output: Rows with a column named `url` holding product URLs.

3. **Add SplitInBatches Node**  
   - Name: `2. Loop Through Each URL`  
   - Purpose: Process each URL individually to avoid overload  
   - Default batch size (1) is suitable.

4. **Add HTTP Request Node for Scrape.do API**  
   - Name: `3. Scrape Product Page HTML`  
   - Method: GET  
   - URL: Use expression:  
     `https://api.scrape.do/?token={{$vars.SCRAPEDO_TOKEN}}&url={{ encodeURIComponent($json.url) }}&geoCode=us&render=false`  
   - Timeout: 60 seconds  
   - Variables: Create a workflow variable `SCRAPEDO_TOKEN` with your Scrape.do API token.

5. **Add HTML Extractor Node**  
   - Name: `4. Extract Raw Data Elements`  
   - Operation: Extract HTML content  
   - Extraction values (key-value pairs):  
     * productTitle: `#productTitle, h1[data-automation-id="product-title"], .product-title`  
     * price: `.a-price .a-offscreen, .a-price-whole, .a-price-fraction, .priceToPay .a-price .a-offscreen`  
     * rating: `.a-icon-alt, [data-hook="average-star-rating"], .a-star-medium .a-icon-alt`  
     * reviewCount: `[data-hook="total-review-count"], .a-link-normal[href*="customerReviews"], #acrCustomerReviewText`  
     * featureBullets: `#feature-bullets ul, .a-unordered-list.a-nostyle.a-vertical.feature`  
     * productDescription: `#productDescription, [data-feature-name="productDescription"], .product-description`  

6. **Set up Langchain Structured Output Parser Node**  
   - Name: `Structured Output Parser`  
   - Schema Type: Manual JSON Schema defining product fields:  
     ```json
     {
       "type": "object",
       "properties": {
         "name": { "type": "string", "description": "Product name/title" },
         "description": { "type": "string", "description": "Product description or key features" },
         "rating": { "type": ["number", "null"], "description": "Average rating (e.g., 4.5)" },
         "reviews": { "type": ["integer", "null"], "description": "Number of reviews" },
         "price": { "type": ["string", "null"], "description": "Product price with currency" }
       },
       "required": ["name"]
     }
     ```

7. **Add Langchain OpenAI Chat Model Node**  
   - Name: `OpenAI Chat Model`  
   - Model: `gpt-4o-mini` (or appropriate GPT-4 model)  
   - Max Tokens: 500  
   - Temperature: 0 for deterministic output  
   - Response Format: JSON object  
   - Credentials: Configure OpenAI or OpenRouter API key.

8. **Add Langchain Chain LLM Node**  
   - Name: `5. Clean & Structure Data with AI`  
   - Prompt Type: Define prompt with message instructing extraction and formatting of product data. Use expression to pass JSON string of raw extracted data.  
   - Connect AI output parser to the Structured Output Parser node.

9. **Add SplitOut Node**  
   - Name: `6. Format Final JSON Output`  
   - Field to split out: `output` (the AI JSON)  
   - Fields to include: `output.name`, `output.description`, `output.rating`, `output.reviews`, `output.price`

10. **Add Google Sheets Node to Append Data**  
    - Name: `7. Save Product Data to Google Sheets`  
    - Operation: Append rows  
    - Document ID: Same as input sheet  
    - Sheet Name: Second tab for results (e.g., `gid=838351250` or Sheet2)  
    - Auto map input fields to columns  
    - Credentials: Google Sheets OAuth2

11. **Connect Nodes in Order:**  
    - Manual Trigger → Get URLs → SplitInBatches → HTTP Request (Scrape.do) → HTML Extractor → Chain LLM → SplitOut → Append to Google Sheets → Loop back to SplitInBatches for next URL.

12. **Set Workflow Variables:**  
    - `SCRAPEDO_TOKEN` with your Scrape.do API token.  
    - Google Sheets OAuth2 credentials configured and authorized.

13. **Google Sheets Setup:**  
    - Create a Google Sheet with two tabs:  
      * Tab 1: List of Amazon product URLs with header `url`  
      * Tab 2: Empty tab to store results with columns: name, description, rating, reviews, price  
    - Share sheet with your Google service account email.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                           | Context or Link                       |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------|
| ## Amazon Scraper with Scrape.do API<br>1. Get Scrape.do API Token at https://scrape.do<br>2. Setup workflow variables (SCRAPEDO_TOKEN, sheet IDs)<br>3. Google Sheets with two tabs: URLs and results<br>4. Add Google Sheets OAuth2 and OpenRouter API credentials<br>5. Advantages over proxy rotation: CAPTCHA handling, reliability | Sticky Note1 content in workflow    |
| Scrape.do API is preferred for Amazon scraping to bypass anti-bot protections without complex proxy setups. It automatically manages CAPTCHA challenges and offers higher success rates compared to services like BrightData.                                                                                           | https://scrape.do                   |

---

**Disclaimer:** The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.

---