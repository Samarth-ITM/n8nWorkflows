Generate Shopify Product Descriptions with GPT-4o Vision & Claude 3.5 + Sales Analytics

https://n8nworkflows.xyz/workflows/generate-shopify-product-descriptions-with-gpt-4o-vision---claude-3-5---sales-analytics-8562


# Generate Shopify Product Descriptions with GPT-4o Vision & Claude 3.5 + Sales Analytics

### 1. Workflow Overview

This workflow automates the generation of Shopify product descriptions for footwear using AI models (GPT-4o Vision and Claude 3.5), combined with sales analytics and error monitoring. It is designed for premium footwear products, focusing on generating SEO-friendly, refined descriptions based on images and product metadata, while managing pagination, filtering, and error logging.

The workflow is logically divided into the following blocks:

- **1.1 Product Data Retrieval and Pagination Handling**  
  Fetch Shopify products with pagination, filter products requiring descriptions, and manage continuation state.

- **1.2 Image Analysis via GPT-4o Vision**  
  Analyze product images to extract observable physical attributes without color or branding inference.

- **1.3 Shopify Product Description Generation (Main Agent)**  
  Use Claude 3.5 via OpenRouter to generate product descriptions based on analyzed image data and product metadata following strict formatting and style rules.

- **1.4 Output Processing and Google Sheets Integration**  
  Update Google Sheets with generated descriptions and maintain processing state and pagination info.

- **1.5 Sales Analytics Reporting**  
  Daily sales data retrieval from Shopify orders, summarization, and logging into Google Sheets.

- **1.6 Error Handling and Notification**  
  Capture workflow errors, analyze them with an AI agent using Perplexity for research, and log detailed explanations into a Google Sheet.

- **1.7 Supporting Utilities and Scheduling**  
  Schedule triggers for hourly and every 5-minute runs, memory buffers for session context, and helper nodes for data filtering and setting.

---

### 2. Block-by-Block Analysis

#### 1.1 Product Data Retrieval and Pagination Handling

- **Overview:**  
  Retrieves Shopify products via API with pagination, filters these to find products needing descriptions (empty `body_html`, appropriate tags, image present), and manages pagination state in Google Sheets.

- **Nodes Involved:**  
  `Schedule Trigger`, `Get row(s) in sheet1`, `Limit1`, `Fetch Shopify Products`, `Code1`, `If3`, `Append row in sheet2`, `Limit2`, `Code5`, `If2`, `Edit Fields1`, `Update row in sheet1`

- **Node Details:**

  - **Schedule Trigger**  
    - Triggers workflow hourly to initiate product retrieval.  
    - No special parameters beyond default interval.  
    - Outputs to `Get row(s) in sheet1`.

  - **Get row(s) in sheet1**  
    - Reads processing state from Google Sheets to get last pagination info.  
    - Uses filters to retrieve relevant rows from a sheet named "ProcessingState".  
    - Outputs to `Limit1`.

  - **Limit1**  
    - Limits the number of product entries processed at once to 10 for manageable batch size.

  - **Fetch Shopify Products**  
    - HTTP Request node configured to Shopify API `products.json` endpoint with API version 2024-04.  
    - Uses OAuth credentials for Shopify.  
    - Query parameters include `limit=200` and dynamic `page_info` for pagination.  
    - Full response enabled for header inspection.

  - **Code1** (JavaScript)  
    - Filters fetched products: must have valid image, empty `body_html`, and tag `currSeas:SS2025`.  
    - Extracts tags for `x-styleCode`, `country_of_origin`, and `gender`.  
    - Avoids duplicates.  
    - Outputs filtered product list with necessary metadata and status "Ready for AI Description".

  - **If3**  
    - Filters out products without a valid `Product ID` to prevent processing invalid data.

  - **Append row in sheet2**  
    - Appends filtered product data into a "Products" sheet in Google Sheets for further processing.

  - **Limit2**  
    - Limits the number of products appended in batch (default no limit specified, likely default 10).

  - **Code5** (JavaScript)  
    - Extracts `page_info` for next pagination from Shopify API response headers (`Link` header).  
    - Handles multiple links and extracts the `next` page info parameter robustly.  
    - Outputs pagination info and debug data.

  - **If2**  
    - Checks if there is a next page (`has_next_page` flag). If true, continues pagination.

  - **Edit Fields1**  
    - Prepares update data for pagination state: increments batch number and sets `page_info`.

  - **Update row in sheet1**  
    - Updates the "ProcessingState" Google Sheet with new pagination info and batch number to track progress.

- **Edge Cases / Failures:**  
  - API rate limits or Shopify connection failure.  
  - Missing or malformed `Link` header causing pagination failure.  
  - Empty product list or no products matching filter criteria.  
  - Duplicate products causing redundant processing.

---

#### 1.2 Image Analysis via GPT-4o Vision

- **Overview:**  
  Analyzes product images using OpenAI's GPT-4o Vision model to extract descriptive physical attributes strictly based on visible features.

- **Nodes Involved:**  
  `Limit1`, `Analyze image`, `Edit Fields`

- **Node Details:**

  - **Analyze image**  
    - OpenAI node configured to use GPT-4o Vision (`modelId: gpt-4o`).  
    - Input: product image URL from filtered products.  
    - Prompt instructs model to describe only clear visible physical footwear attributes (adjustability, closure, heel, materials, sole, toe shape, cushioning).  
    - Excludes color, logos, or branding.  
    - Outputs a short descriptive summary (~50 words).

  - **Edit Fields**  
    - Transfers image description to a new field "content".  
    - Passes forward product metadata from `Limit1` node.

- **Edge Cases / Failures:**  
  - Image URL invalid or inaccessible.  
  - GPT-4o Vision API request timeout or quota exceeded.  
  - Model returns empty or irrelevant description.

---

#### 1.3 Shopify Product Description Generation (Main Agent)

- **Overview:**  
  Uses Claude 3.5 (Anthropic) via OpenRouter to generate SEO-optimized, benefit-led Shopify product descriptions for footwear, based on image analysis and product metadata.

- **Nodes Involved:**  
  `Edit Fields`, `Shopify Content Generator`, `Structured Output Parser`, `OpenRouter Chat Model`, `OpenRouter Chat Model1`, `Message a model in Perplexity`

- **Node Details:**

  - **Shopify Content Generator**  
    - LangChain agent node configured with Claude 3.5 model via OpenRouter.  
    - Inputs a structured prompt including product ID, title, type, vendor, image description, status, country of origin, style code, and gender.  
    - System message specifies tone, style, SEO rules, formatting, strict exclusion of color, and origin/design country logic.  
    - Outputs JSON with fields: product_id, product_title (cleaned), generated_description (60-80 words), status ("generated" or "retry").  
    - Has output parser configured to enforce structured JSON output.

  - **Structured Output Parser**  
    - Parses AI output to validate and correct JSON structure automatically.

  - **OpenRouter Chat Model** & **OpenRouter Chat Model1**  
    - Serve as AI language models for the agent and parser nodes, respectively.

  - **Message a model in Perplexity**  
    - Optional AI tool integration to query external knowledge base if needed to confirm country of design or other missing info.

- **Edge Cases / Failures:**  
  - AI output malformed or missing required fields.  
  - Failure to confidently confirm origin/design country triggers retry status.  
  - API quota limits or connectivity issues with OpenRouter or OpenAI.  
  - Inconsistent or conflicting input data causing generation failure.

---

#### 1.4 Output Processing and Google Sheets Integration

- **Overview:**  
  Updates the Google Sheets "Products" sheet with the generated description and status, and maintains data consistency.

- **Nodes Involved:**  
  `Shopify Content Generator`, `Update row in sheet`

- **Node Details:**

  - **Update row in sheet**  
    - Updates product row in Google Sheets with generated description, product title, status.  
    - Matches by `Product ID` to ensure correct row update.

- **Edge Cases / Failures:**  
  - Google Sheets API errors or permission issues.  
  - Mismatched or missing Product ID leading to failed update.  
  - Data type or format mismatches preventing row update.

---

#### 1.5 Sales Analytics Reporting

- **Overview:**  
  Retrieves daily Shopify sales orders, sums total sales, and records results in a Google Sheet as a daily report.

- **Nodes Involved:**  
  `Schedule Trigger1`, `HTTP Request1`, `If`, `Split Out`, `Edit Fields2`, `Summarize`, `Append row in sheet`, `Append row in sheet1`, `Sticky Note5`

- **Node Details:**

  - **Schedule Trigger1**  
    - Runs daily at 14:01 to start sales data retrieval.

  - **HTTP Request1**  
    - Requests Shopify orders API filtered for the previous day, financial status "paid".  
    - Uses OAuth credentials for Shopify.

  - **If**  
    - Checks if any orders were returned.

  - **Split Out**  
    - Splits orders array to process individual orders.

  - **Edit Fields2**  
    - Extracts `current_total_price` from each order for summation.

  - **Summarize**  
    - Aggregates total sales by summing `current_price` across orders.

  - **Append row in sheet**  
    - Logs daily total sales and date into a Google Sheet "shopify_daily_sales".

  - **Append row in sheet1**  
    - If no orders found, logs zero sales for the day.

- **Edge Cases / Failures:**  
  - Shopify API rate limits or failures.  
  - No orders found for the day triggering zero sales entry.  
  - Google Sheets permission or API errors.

---

#### 1.6 Error Handling and Notification

- **Overview:**  
  Monitors workflow errors, diagnoses root cause using an AI agent, researches via Perplexity, and logs detailed error explanations to a Google Sheet.

- **Nodes Involved:**  
  `Error Trigger`, `AI Agent`, `Message a model in Perplexity1`, `Append row in sheet in Google Sheets`, `Sticky Note1`

- **Node Details:**

  - **Error Trigger**  
    - Listens for any errors in the workflow execution.

  - **AI Agent**  
    - Receives error message and metadata.  
    - Diagnoses cause with embedded instructions.  
    - Uses Perplexity tool to research unclear errors.  
    - Crafts clear explanation with layman and technical insights.  
    - Outputs structured string for logging.

  - **Message a model in Perplexity1**  
    - Executes Perplexity queries as part of AI Agent research.

  - **Append row in sheet in Google Sheets**  
    - Appends error explanation and timestamp to "Error_log" Google Sheet.

- **Edge Cases / Failures:**  
  - Failure in error analysis node itself.  
  - Perplexity API quota or connection issues.  
  - Google Sheets logging failures.

---

#### 1.7 Supporting Utilities and Scheduling

- **Overview:**  
  Provides scheduling triggers and session memory buffers for AI conversation context and process pacing.

- **Nodes Involved:**  
  `Every 5 Minutes`, `Get row(s) in sheet2`, `Simple Memory`, `Session Memory2`, `When clicking ‘Execute workflow’`, `Sticky Note`, `Sticky Note2`, `Sticky Note6`, `Sticky Note5`, `Sticky Note1`, `Sticky Note`

- **Node Details:**

  - **Every 5 Minutes**  
    - Triggers product filtering and processing every 5 minutes.

  - **Get row(s) in sheet2**  
    - Fetches products queued for AI description generation.

  - **Simple Memory** & **Session Memory2**  
    - LangChain memory buffer nodes to maintain session context keyed by current minute for AI agents.

  - **When clicking ‘Execute workflow’**  
    - Manual trigger for testing or manual execution.

  - **Sticky Notes**  
    - Provide documentation and notes for pagination, error notification, tag filtering, and sales report.

- **Edge Cases / Failures:**  
  - Trigger misfires or scheduler delays.  
  - Memory buffers not maintained properly leading to context loss.

---

### 3. Summary Table

| Node Name                    | Node Type                                      | Functional Role                                  | Input Node(s)                  | Output Node(s)                   | Sticky Note                                                                                               |
|------------------------------|------------------------------------------------|------------------------------------------------|--------------------------------|---------------------------------|-----------------------------------------------------------------------------------------------------------|
| Schedule Trigger              | Schedule Trigger                               | Hourly trigger to start product retrieval       |                                | Get row(s) in sheet1             |                                                                                                           |
| Get row(s) in sheet1          | Google Sheets                                  | Fetch processing state (pagination info)        | Schedule Trigger              | Limit1                          |                                                                                                           |
| Limit1                       | Limit                                          | Limit batch size of products                      | Get row(s) in sheet1          | Analyze image                   |                                                                                                           |
| Fetch Shopify Products        | HTTP Request                                   | Fetch products from Shopify API                   | Get row(s) in sheet2          | Code1, Code5                   | Fixed: API 2024-04, fullResponse enabled, conditional pagination                                          |
| Code1                        | Code                                           | Filter products needing descriptions              | Fetch Shopify Products        | If3                            |                                                                                                           |
| If3                          | If                                             | Check valid Product ID                             | Code1                        | Append row in sheet2            |                                                                                                           |
| Append row in sheet2          | Google Sheets                                  | Append filtered product data to "Products" sheet | If3                          | Limit2                         |                                                                                                           |
| Limit2                       | Limit                                          | Limit number of products processed per batch     | Append row in sheet2          |                               |                                                                                                           |
| Code5                        | Code                                           | Extract and parse pagination `page_info` header  | Fetch Shopify Products        | If2                            |                                                                                                           |
| If2                          | If                                             | Check if next page exists                         | Code5                        | Edit Fields1                   |                                                                                                           |
| Edit Fields1                 | Set                                            | Prepare pagination state update                   | If2                          | Update row in sheet1            |                                                                                                           |
| Update row in sheet1          | Google Sheets                                  | Update pagination info in "ProcessingState" sheet| Edit Fields1                 |                               |                                                                                                           |
| Analyze image                | OpenAI (GPT-4o Vision)                         | Analyze product image for physical attributes    | Limit1                       | Edit Fields                    |                                                                                                           |
| Edit Fields                  | Set                                            | Prepare data with image description               | Analyze image                | Shopify Content Generator       |                                                                                                           |
| Shopify Content Generator     | LangChain Agent (Claude 3.5 via OpenRouter)   | Generate product descriptions                      | Edit Fields                  | Update row in sheet             | FIXED: Output format to match sheets mapping                                                              |
| Structured Output Parser      | LangChain Output Parser                        | Parse AI-generated JSON output                     | OpenRouter Chat Model1        | Shopify Content Generator       |                                                                                                           |
| OpenRouter Chat Model         | LangChain Model                                | AI language model for main content generation     |                              | Shopify Content Generator       |                                                                                                           |
| OpenRouter Chat Model1        | LangChain Model                                | AI language model for output parsing               |                              | Structured Output Parser        |                                                                                                           |
| OpenRouter Chat Model2        | LangChain Model                                | AI language model for error analysis agent        |                              | AI Agent                       |                                                                                                           |
| Message a model in Perplexity | Perplexity Tool                                | External knowledge lookup for content generation  |                              | Shopify Content Generator       |                                                                                                           |
| Message a model in Perplexity1| Perplexity Tool                                | External knowledge lookup for error analysis      |                              | AI Agent                       |                                                                                                           |
| Update row in sheet           | Google Sheets                                  | Update product row with generated description     | Shopify Content Generator    |                               |                                                                                                           |
| Append row in sheet in Google Sheets | Google Sheets Tool                       | Append error logs to error sheet                   | AI Agent                     |                               |                                                                                                           |
| Error Trigger                | Error Trigger                                  | Catch workflow errors                              |                              | AI Agent                      | # Error Notification \n\n###  If there is any error related to Api or server error mainly it will notify the owner about the type fo the error and explain the reason |
| AI Agent                    | LangChain Agent                                | Analyze errors, research, explain and log them   | Error Trigger                | Append row in sheet in Google Sheets |                                                                                                           |
| Simple Memory               | LangChain MemoryBufferWindow                   | Context memory for AI agents                       |                              | AI Agent                      |                                                                                                           |
| Session Memory2             | LangChain MemoryBufferWindow                   | Context memory for Shopify Content Generator      |                              | Shopify Content Generator       |                                                                                                           |
| Every 5 Minutes             | Cron Trigger                                   | 5-minute trigger to check for new products        |                              | Get row(s) in sheet2            |                                                                                                           |
| Get row(s) in sheet2         | Google Sheets                                  | Get queued products for description generation    | Every 5 Minutes              | Fetch Shopify Products          |                                                                                                           |
| Append row in sheet1          | Google Sheets                                  | Append sales data row (zero sales if no orders)   | If                          |                               |                                                                                                           |
| HTTP Request1                | HTTP Request                                   | Retrieve Shopify orders for sales analytics       | Schedule Trigger1            | If                            |                                                                                                           |
| If                          | If                                             | Check if orders exist                              | HTTP Request1                | Split Out, Append row in sheet1 |                                                                                                           |
| Split Out                   | Split Out Node                                 | Split orders array for processing                  | If                          | Edit Fields2                   |                                                                                                           |
| Edit Fields2                | Set                                            | Prepare sales price field for summarization       | Split Out                   | Summarize                     |                                                                                                           |
| Summarize                  | Summarize Node                                 | Sum total sales                                    | Edit Fields2                | Append row in sheet            |                                                                                                           |
| Append row in sheet          | Google Sheets                                  | Append daily sales data                            | Summarize                   |                               |                                                                                                           |
| Sticky Note                 | Sticky Note                                    | Shopify Product Description agent overview         |                              |                               | # Shopify Product Description agent \n\n\n## Pagination Handling\n\n\n1. schedule every hour \n2. Search if there has already been run from the previous attempt or not if there has been previous run it will start from the latest product id.\n\n3. Code node will filter the products fetched each time to check whether it has body_html , CurrSeas:SS2025 or it has image then it process \n\n4. The Open Ai image node will do the descriptions of image based on the image link. and feed it to the main agent\n\n5. The main agent or the content creator will write the product description according to the AU format.\n\n6. The output format for the main agent is mainly all the fields id,vendor,generated description, status etc.\n\n7. The second code node keeps record of next page url to continue from there and record it in the sheet.\n\n |
| Sticky Note1                | Sticky Note                                    | Error notification overview                        |                              |                               | # Error Notification \n\n\n###  If there is any error related to Api or server error mainly it will notify the owner about the type fo the error and explain the reason\n\n |
| Sticky Note2                | Sticky Note                                    | Pagination Handler note                            |                              |                               | ## Pagination Handler\n\nworking code\n                                                                                                         |
| Sticky Note5                | Sticky Note                                    | Shopify Daily Sales report                         |                              |                               | ## Shopify Daily Sales report\n                                                                                                               |
| Sticky Note6                | Sticky Note                                    | Tag filtration note                               |                              |                               | ## Tag filteration\n                                                                                                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node (Hourly)**  
   - Set to trigger every hour.  
   - Connect to "Get row(s) in sheet1".

2. **Create Google Sheets node "Get row(s) in sheet1"**  
   - Connect to your Google Sheets account.  
   - Configure to read the "ProcessingState" sheet, filtering relevant rows for pagination status.

3. **Add a Limit node "Limit1"**  
   - Limit to 10 items to batch process products.

4. **Create Google Sheets node "Get row(s) in sheet2"**  
   - Reads product rows queued for processing from the "Products" sheet.

5. **Create HTTP Request node "Fetch Shopify Products"**  
   - Use Shopify Admin API (2024-04) `products.json` endpoint.  
   - Add OAuth credentials for Shopify.  
   - Query parameters: `limit=200`, `page_info` from sheet state.  
   - Enable full response for access to headers.

6. **Create a JavaScript Code node "Code1"**  
   - Filter products with:  
     - Non-empty image URL  
     - Empty `body_html`  
     - Tag `currSeas:SS2025` present  
   - Extract tags for `x-styleCode`, `country_of_origin`, and `gender`.  
   - Remove duplicates.

7. **Create an If node "If3"**  
   - Condition: `Product ID` is not empty.  
   - True path connects to "Append row in sheet2".

8. **Create Google Sheets node "Append row in sheet2"**  
   - Append filtered products to "Products" sheet.

9. **Create a Limit node "Limit2"** (optional)  
   - To limit batch size for appending rows.

10. **Create a JavaScript Code node "Code5"**  
    - Parse `Link` header from Shopify response to extract `page_info` for next page.  
    - Handle multiple links and fallback extraction.  
    - Output pagination info.

11. **Create If node "If2"**  
    - Check if `has_next_page` is true.  
    - True path connects to "Edit Fields1".

12. **Create Set node "Edit Fields1"**  
    - Increment batch number from sheet.  
    - Store new `page_info`.

13. **Create Google Sheets node "Update row in sheet1"**  
    - Update "ProcessingState" with new pagination info and batch number.

14. **Create OpenAI node "Analyze image"**  
    - Use GPT-4o Vision model.  
    - Input: product image URL.  
    - Prompt instructs description of visible physical footwear attributes, excluding color/branding.

15. **Create Set node "Edit Fields"**  
    - Add image analysis output to product data fields.

16. **Create LangChain Agent node "Shopify Content Generator"**  
    - Use Claude 3.5 model via OpenRouter API.  
    - Input: product metadata + image description.  
    - System message: detailed copywriting instructions with strict style and format rules.  
    - Output: structured JSON with product description and status.  
    - Attach Structured Output Parser node to validate JSON.

17. **Create Google Sheets node "Update row in sheet"**  
    - Update product row with generated description and status.

18. **Create error handling nodes:**  
    - Error Trigger node to catch errors.  
    - LangChain Agent node "AI Agent" with a detailed prompt to diagnose errors.  
    - Perplexity tool node for external research.  
    - Google Sheets node to append error details to "Error_log".

19. **Create nodes for sales analytics:**  
    - Schedule Trigger1: daily at 14:01.  
    - HTTP Request1: fetch Shopify orders filtered by day and financial status.  
    - If node: check if orders exist.  
    - Split Out: split orders array.  
    - Set node: extract order price.  
    - Summarize node: sum prices.  
    - Append row in sheet: log daily sales.

20. **Create memory buffer nodes:**  
    - LangChain memory buffer nodes (`Simple Memory`, `Session Memory2`) to maintain AI session context keyed by minute.

21. **Add manual trigger and sticky notes for documentation.**

22. **Connect nodes as per dependencies described above, ensuring proper credential assignment:**  
    - OpenAI API credentials for GPT-4o Vision.  
    - OpenRouter API credentials for Claude 3.5.  
    - Shopify OAuth credentials.  
    - Google Sheets OAuth credentials.  
    - Perplexity API credentials.

---

### 5. General Notes & Resources

| Note Content                                                                                                                   | Context or Link                                                                                                 |
|-------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------|
| Shopify Product Description agent - Pagination handling and description process overview.                                        | See Sticky Note node "Sticky Note" for detailed explanation.                                                  |
| Error Notification mechanism informs owner with error type and explanation.                                                     | See Sticky Note node "Sticky Note1".                                                                           |
| Pagination handler code is robust and tested for multiple link headers from Shopify.                                            | See Sticky Note node "Sticky Note2".                                                                           |
| Shopify Daily Sales report is generated daily and logged to Google Sheets.                                                     | See Sticky Note node "Sticky Note5".                                                                           |
| Tag filtration logic filters products having specific tags and empty descriptions for processing.                              | See Sticky Note node "Sticky Note6".                                                                           |
| Workflow uses Australian English and targets modern women aged 60+ for product descriptions, adjusting tone if product is men’s.| System prompt in "Shopify Content Generator".                                                                  |
| External research with Perplexity is integrated into both content generation and error analysis to ensure accuracy and clarity. | Integrated in "Message a model in Perplexity" nodes.                                                           |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This process strictly complies with current content policies and contains no illegal, offensive, or protected elements. All processed data is legal and publicly available.