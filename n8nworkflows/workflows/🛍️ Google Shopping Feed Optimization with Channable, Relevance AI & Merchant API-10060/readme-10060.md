🛍️ Google Shopping Feed Optimization with Channable, Relevance AI & Merchant API

https://n8nworkflows.xyz/workflows/----google-shopping-feed-optimization-with-channable--relevance-ai---merchant-api-10060


# 🛍️ Google Shopping Feed Optimization with Channable, Relevance AI & Merchant API

### 1. Workflow Overview

This workflow automates the daily optimization and sync of a Google Shopping product feed using Channable, Relevance AI, and the Google Merchant API. It is designed for e-commerce businesses aiming to improve product data quality, enhance titles and descriptions via AI, assign strategic custom labels for bidding segmentation, and maintain up-to-date product listings on Google Merchant Center with automated error monitoring and alerting.

The workflow is logically divided into four main blocks:

- **1.1 Input Reception and Quality Validation:** Triggered daily at 6 AM, it retrieves the raw product feed and validates key data points (GTINs, images, categories, prices, titles, descriptions, availability) assigning quality scores and marking products needing optimization.

- **1.2 AI-Driven Feed Optimization:** Splits the full feed into individual products, then sequentially optimizes each product’s title and description using Relevance AI tools. It further assigns five custom labels based on margin, performance, seasonality, stock, and category before aggregating the optimized items back into a single dataset.

- **1.3 Merchant Center Synchronization and Status Check:** Uploads the optimized feed in batches to Google Merchant Center via the updated Merchant API (v2.1), then queries the Merchant API to check product status, capturing disapprovals and warnings.

- **1.4 Alerting and Reporting:** Based on product status, it routes the workflow to send either an immediate Slack alert if disapprovals exist or a success summary notification, ensuring the team is informed about feed health daily.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Input Reception and Quality Validation

**Overview:** This block triggers the workflow daily, fetches the latest product feed from Channable (or any e-commerce API), performs data quality validations on each product, and prepares the feed for AI optimization.

**Nodes Involved:**  
- Daily Trigger - 6 AM  
- Get Product Feed  
- Data Quality Checks  
- Split Products1

**Node Details:**

- **Daily Trigger - 6 AM**  
  - *Type:* Schedule Trigger  
  - *Role:* Initiates the workflow every day at 6:00 AM, ensuring timely feed optimization before Merchant Center sync.  
  - *Configuration:* Cron expression `"0 6 * * *"` for daily execution at 6 AM.  
  - *Connections:* Outputs to "Get Product Feed".  
  - *Edge Cases:* Missed triggers if n8n instance is down; time zone considerations for cron expression.  

- **Get Product Feed**  
  - *Type:* HTTP Request  
  - *Role:* Retrieves the product feed JSON from Channable API or alternate e-commerce API endpoint.  
  - *Configuration:* URL uses environment variables for API URL, company, project, and feed IDs. Authenticated via HTTP header credentials.  
  - *Inputs:* Triggered by "Daily Trigger - 6 AM".  
  - *Outputs:* Sends feed data to "Data Quality Checks".  
  - *Edge Cases:* API downtime, authentication failures, empty or malformed feed.  

- **Data Quality Checks**  
  - *Type:* Code (JavaScript)  
  - *Role:* Validates product attributes such as GTIN, images, categories, pricing, title and description lengths, availability; calculates a quality score and flags issues.  
  - *Configuration:* Custom JS script processing all products at once. Outputs categorized products (valid and issues) and overall counts.  
  - *Inputs:* Receives feed JSON array from "Get Product Feed".  
  - *Outputs:* Sends enriched product array to "Split Products1".  
  - *Edge Cases:* Unexpected product schema, missing fields, zero products.  

- **Split Products1**  
  - *Type:* Item Lists  
  - *Role:* Splits the validated product array into individual product items for parallel AI optimization and labeling.  
  - *Configuration:* Splits on field `all_products` from previous node output.  
  - *Inputs:* From "Data Quality Checks".  
  - *Outputs:* Routes each product individually to "Optimize Title".  
  - *Edge Cases:* Empty input array, splitting errors.  

---

#### 2.2 AI-Driven Feed Optimization

**Overview:** Processes each product individually to optimize titles and generate improved descriptions using Relevance AI, then assigns meaningful custom labels to support segmented Google Ads bidding strategies. Finally, aggregates all optimized products back to a single list.

**Nodes Involved:**  
- Optimize Title  
- Generate Description  
- Assign Custom Labels  
- Aggregate Products

**Node Details:**

- **Optimize Title**  
  - *Type:* HTTP Request  
  - *Role:* Calls the Relevance AI title optimization tool API to enhance product titles with structured brand, type, features, and attributes, respecting length constraints (max 150 chars).  
  - *Configuration:* Uses environment variables for API URL and tool ID; sends product fields (current title, category, brand, color, size, material) in JSON body. Uses HTTP header auth credentials.  
  - *Inputs:* Product items from "Split Products1".  
  - *Outputs:* Optimized title sent to "Generate Description".  
  - *Edge Cases:* API timeouts, missing fields, malformed responses.  

- **Generate Description**  
  - *Type:* HTTP Request  
  - *Role:* Calls Relevance AI description generation tool to create 300-400 character benefit-focused descriptions based on optimized title or original title, product features, and category.  
  - *Configuration:* Similar to "Optimize Title", uses respective tool ID and environment variables.  
  - *Inputs:* Receives optimized titles from "Optimize Title".  
  - *Outputs:* Sends enriched product with generated descriptions to "Assign Custom Labels".  
  - *Edge Cases:* API errors, empty input fields.  

- **Assign Custom Labels**  
  - *Type:* Code (JavaScript)  
  - *Role:* Assigns five custom labels for Google Shopping campaign segmentation: margin tier, performance status, seasonality, stock level, and category short code. Also recalculates margin percentage.  
  - *Configuration:* Custom JS logic using product pricing, cost, sales rank, availability, quantity, and category fields.  
  - *Inputs:* Receives products with optimized title/description from "Generate Description".  
  - *Outputs:* Sends labeled products to "Aggregate Products".  
  - *Edge Cases:* Missing numeric fields, improper date formats, zero or negative pricing.  

- **Aggregate Products**  
  - *Type:* Aggregate  
  - *Role:* Combines individually processed products back into a single array to prepare for batch upload.  
  - *Configuration:* Default aggregation of all item data.  
  - *Inputs:* From "Assign Custom Labels".  
  - *Outputs:* Sends aggregated feed to "Upload to Merchant Center (Content API v2.1)".  
  - *Edge Cases:* Empty input set, aggregation failures.  

---

#### 2.3 Merchant Center Synchronization and Status Check

**Overview:** Uploads the optimized feed in batches to Google Merchant Center using the latest Merchant API, then fetches product status to detect disapprovals or warnings.

**Nodes Involved:**  
- Upload to Merchant Center (Content API v2.1)  
- Check Product Status (NEW API)  
- Analyze Product Issues  
- IF Disapprovals Found

**Node Details:**

- **Upload to Merchant Center (Content API v2.1)**  
  - *Type:* HTTP Request  
  - *Role:* Publishes optimized products in batches (up to 1000 per batch) to Google Merchant Center via the new Merchant API endpoint `/products`.  
  - *Configuration:* POST method, JSON payload with all product fields including custom labels, authenticated with Google OAuth2 credentials. Batch size limited to 1000.  
  - *Inputs:* Aggregated product list from "Aggregate Products".  
  - *Outputs:* Triggers "Check Product Status (NEW API)" after upload.  
  - *Edge Cases:* API rate limits, authentication expiration, batch upload failures, malformed product data.  

- **Check Product Status (NEW API)**  
  - *Type:* HTTP Request  
  - *Role:* Retrieves the current product listings from Merchant Center to check for disapprovals and warnings. Uses Merchant API with OAuth2 credentials.  
  - *Configuration:* GET request to `/products` endpoint with paging to handle up to 1000 products per request.  
  - *Inputs:* Triggered after upload completion.  
  - *Outputs:* Sends product status data to "Analyze Product Issues".  
  - *Edge Cases:* API pagination, network errors, auth token expiration.  

- **Analyze Product Issues**  
  - *Type:* Code (JavaScript)  
  - *Role:* Parses Merchant API response to identify and categorize product disapprovals and warnings by severity, counting occurrences and preparing issue lists for alerting.  
  - *Configuration:* Processes `products` array with nested `issues`, checking severity and applicability flags.  
  - *Inputs:* Response from "Check Product Status (NEW API)".  
  - *Outputs:* Sends counts and issue lists to "IF Disapprovals Found".  
  - *Edge Cases:* Unexpected API response formats, empty or missing fields.  

- **IF Disapprovals Found**  
  - *Type:* IF Node  
  - *Role:* Conditional routing based on whether any critical disapprovals are detected (disapproval_count > 0).  
  - *Configuration:* Checks numeric condition against zero.  
  - *Inputs:* Receives analyzed issue data from "Analyze Product Issues".  
  - *Outputs:* Routes to "Alert - Disapprovals" if true; else "Success Summary".  
  - *Edge Cases:* Null or missing disapproval count.  

---

#### 2.4 Alerting and Reporting

**Overview:** Sends Slack notifications either alerting the team of critical disapprovals requiring immediate attention or confirming successful daily feed optimization.

**Nodes Involved:**  
- Alert - Disapprovals  
- Success Summary

**Node Details:**

- **Alert - Disapprovals**  
  - *Type:* Slack  
  - *Role:* Sends a detailed Slack message alert listing up to 10 disapproved products with their issues, highlighting critical problems that require review in Merchant Center.  
  - *Configuration:* Uses Slack webhook with custom message formatting including counts and issue details.  
  - *Inputs:* From "IF Disapprovals Found" true branch.  
  - *Outputs:* Ends workflow.  
  - *Edge Cases:* Slack webhook failures, message formatting errors.  

- **Success Summary**  
  - *Type:* Slack  
  - *Role:* Posts a daily summary message to Slack indicating total products processed, number of quality issues, disapprovals, warnings, and confirmation of successful feed optimization.  
  - *Configuration:* Slack webhook with summary text referencing node outputs and timestamps.  
  - *Inputs:* From "IF Disapprovals Found" false branch.  
  - *Outputs:* Ends workflow.  
  - *Edge Cases:* Slack webhook connectivity issues.  

---

### 3. Summary Table

| Node Name                             | Node Type            | Functional Role                                | Input Node(s)                  | Output Node(s)                     | Sticky Note                                                                                  |
|-------------------------------------|----------------------|-----------------------------------------------|--------------------------------|-----------------------------------|----------------------------------------------------------------------------------------------|
| Daily Trigger - 6 AM                 | Schedule Trigger     | Starts workflow daily at 6 AM                  | -                              | Get Product Feed                  | Triggers daily at 6 AM for feed optimization before Merchant Center sync                     |
| Get Product Feed                    | HTTP Request         | Retrieves raw product feed                      | Daily Trigger - 6 AM            | Data Quality Checks              | Retrieves product feed. Can replace with your e-commerce platform API endpoint.             |
| Data Quality Checks                 | Code (JS)            | Validates product data quality, assigns scores | Get Product Feed                | Split Products1                  | Validates: GTINs, images, categories, pricing, title/description length. Assigns quality score. |
| Split Products1                    | Item Lists           | Splits product array into individual items     | Data Quality Checks             | Optimize Title                  | Splits the 'all_products' array from Data Quality Checks into individual product items.      |
| Optimize Title                    | HTTP Request         | AI optimization of product titles              | Split Products1                 | Generate Description            | CORRECTED: Uses /trigger with tool ID. Optimizes title: Brand + Type + Features + Attributes. Max 150 chars. |
| Generate Description              | HTTP Request         | AI-generated product benefit-focused descriptions | Optimize Title                 | Assign Custom Labels            | CORRECTED: Uses /trigger with tool ID. Generates 300-400 char benefit-focused description.  |
| Assign Custom Labels              | Code (JS)            | Assigns 5 custom labels for campaign segmentation | Generate Description            | Aggregate Products              | Assigns 5 custom labels for segmented bidding: margin, performance, seasonality, stock, category |
| Aggregate Products               | Aggregate            | Recombines optimized products into one array   | Assign Custom Labels            | Upload to Merchant Center (Content API v2.1) | Combines all optimized products                                                             |
| Upload to Merchant Center (Content API v2.1) | HTTP Request         | Uploads optimized products to Merchant Center  | Aggregate Products             | Check Product Status (NEW API)   | CRITICAL CORRECTION: Uses NEW Merchant API (merchantapi.googleapis.com) instead of deprecated Content API. Max 1000 products per batch. |
| Check Product Status (NEW API)   | HTTP Request         | Retrieves product status from Merchant Center  | Upload to Merchant Center (Content API v2.1) | Analyze Product Issues          | CORRECTED: Uses NEW Merchant API to list products and check for issues. Supports up to 1000 page size. |
| Analyze Product Issues           | Code (JS)            | Parses status response to identify disapprovals and warnings | Check Product Status (NEW API) | IF Disapprovals Found           | CORRECTED: Parses NEW Merchant API response format for issues                               |
| IF Disapprovals Found            | IF                   | Routes based on presence of critical disapprovals | Analyze Product Issues          | Alert - Disapprovals, Success Summary | Check if any critical disapprovals exist                                                     |
| Alert - Disapprovals             | Slack                | Sends immediate alert if disapprovals detected | IF Disapprovals Found (true)   | -                               | Sends alert if disapprovals found                                                           |
| Success Summary                 | Slack                | Sends daily success summary if no disapprovals | IF Disapprovals Found (false)  | -                               | Sends daily summary of feed optimization                                                   |
| Sticky Note                     | Sticky Note          | Documentation note                              | -                              | -                               | # 🛍️ Google Shopping Feed Optimization with Relevance AI + Google Merchant API\n\nAutomates your Google Shopping product feed optimization. Uses Relevance AI to enhance titles/descriptions, adds custom labels, and syncs optimized products daily to Merchant Center. |
| Sticky Note2                    | Sticky Note          | Documentation note for Stage 1                  | -                              | -                               | ## 🟨 Stage 1 — Workflow Start\n\n| 🕓 Daily Trigger – 6 AM | 📦 Get Product Feed | 🔍 Data Quality Checks | ✂️ Split Products |\n| Runs automatically every morning.<br>Starts 6 AM daily to refresh and optimize the feed before Merchant Center sync. | Fetches product data from Channable (or your e-commerce API). | Checks for missing GTINs, titles, prices, and descriptions.Calculates a quality score for each item. | Prepares each product for AI optimization by splitting the feed into individual items. |
| Sticky Note3                    | Sticky Note          | Documentation note for Stage 2                  | -                              | -                               | ## 🟨 Stage 2 — AI Optimization Stage\n\n| 🧠 Optimize Title | 📝 Generate Description | 🏷️ Assign Custom Labels | 📊 Aggregate Products |\n| AI-powered title optimizer using Relevance AI.Improves clarity and SEO.e.g., “Running Shoes” → “Nike Air Zoom Men’s Running Shoes – Size 42”. | AI-generated 300–400 char benefit-focused descriptions via Relevance AI. | Adds five campaign segmentation labels (margin, performance, seasonality, stock, category). | Combines all optimized products back into a single feed ready for upload. |
| Sticky Note4                    | Sticky Note          | Documentation note for Stage 3                  | -                              | -                               | ## 🟨 Stage 3 — Merchant Center Sync\n\n| 🚀 Upload to Merchant Center (API v2.1) |  🔁 Check Product Status (NEW API) | ⚠️ Analyze Product Issues | 🚦 IF Disapprovals Found |\n| Publishes optimized feed in batches (max 1000) using the new Merchant API (`merchantapi.googleapis.com`). | Verifies upload results by listing live products and checking for warnings or errors. | Detects disapprovals and warnings from Merchant API responses and groups them by severity. | Routes workflow based on disapprovals – alerts team if issues exist or sends success summary otherwise. |
| Sticky Note5                    | Sticky Note          | Documentation note for Stage 4                  | -                              | -                               | ## 🟨 Stage 4 — Notifications\n\n| 🚨 Alert – Disapprovals | ✅ Success Summary |\n| Immediate Slack alert listing disapproved products and issues requiring review. | Posts daily Slack summary with total products, issues, and confirmation of successful optimization. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node: "Daily Trigger - 6 AM"**  
   - Type: Schedule Trigger  
   - Parameters: Cron expression `0 6 * * *` (daily at 6:00 AM)  
   - No credentials needed  

2. **Create HTTP Request Node: "Get Product Feed"**  
   - Type: HTTP Request  
   - URL: Use environment variables `{{$env.CHANNABLE_API_URL}}/companies/{{$env.CHANNABLE_COMPANY_ID}}/projects/{{$env.CHANNABLE_PROJECT_ID}}/feeds/{{$env.FEED_ID}}`  
   - Authentication: HTTP Header Auth (configure credential with required header)  
   - Method: GET (default)  

3. **Connect "Daily Trigger - 6 AM" → "Get Product Feed"**

4. **Create Code Node: "Data Quality Checks"**  
   - Type: Code (JavaScript)  
   - Paste provided JS code that validates GTIN, image link, category, price, title length, description length, availability, and calculates quality scores  
   - No credentials needed  

5. **Connect "Get Product Feed" → "Data Quality Checks"**

6. **Create Item Lists Node: "Split Products1"**  
   - Type: Item Lists  
   - Field to split out: `all_products` (from Data Quality Checks output)  

7. **Connect "Data Quality Checks" → "Split Products1"**

8. **Create HTTP Request Node: "Optimize Title"**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `{{$env.RELEVANCE_AI_API_URL}}/tools/{{$env.RELEVANCE_TOOL_TITLE_OPTIMIZER_ID}}/trigger`  
   - Authentication: HTTP Header Auth (same as Get Product Feed)  
   - Body: JSON with product fields `current_title`, `category`, `brand`, `color`, `size`, `material` from input JSON  
   - Send body as JSON  

9. **Connect "Split Products1" → "Optimize Title"**

10. **Create HTTP Request Node: "Generate Description"**  
    - Type: HTTP Request  
    - Method: POST  
    - URL: `{{$env.RELEVANCE_AI_API_URL}}/tools/{{$env.RELEVANCE_TOOL_DESCRIPTION_ID}}/trigger`  
    - Authentication: HTTP Header Auth  
    - Body: JSON with `product_title` (optimized or original), `product_features` (description), `category`  

11. **Connect "Optimize Title" → "Generate Description"**

12. **Create Code Node: "Assign Custom Labels"**  
    - Type: Code (JavaScript)  
    - Paste JS code assigning custom labels for margin tier, performance, seasonality, stock levels, and category short code based on product data  
    - No credentials needed  

13. **Connect "Generate Description" → "Assign Custom Labels"**

14. **Create Aggregate Node: "Aggregate Products"**  
    - Type: Aggregate  
    - Default aggregation of all item data (combine individual products back into single array)  

15. **Connect "Assign Custom Labels" → "Aggregate Products"**

16. **Create HTTP Request Node: "Upload to Merchant Center (Content API v2.1)"**  
    - Type: HTTP Request  
    - Method: POST  
    - URL: `{{$env.MERCHANT_API_URL}}/{{$env.MERCHANT_ACCOUNT_ID}}/products`  
    - Authentication: Google OAuth2 credential (Google API)  
    - Body: JSON with product fields including optimized title, description, custom labels, and other required Merchant API fields  
    - Enable batching with batch size 1000  

17. **Connect "Aggregate Products" → "Upload to Merchant Center (Content API v2.1)"**

18. **Create HTTP Request Node: "Check Product Status (NEW API)"**  
    - Type: HTTP Request  
    - Method: GET  
    - URL: `{{$env.MERCHANT_API_URL}}/{{$env.MERCHANT_ACCOUNT_ID}}/products`  
    - Authentication: Google OAuth2 credential (Google API)  
    - Support pagination if necessary (up to 1000 per page)  

19. **Connect "Upload to Merchant Center (Content API v2.1)" → "Check Product Status (NEW API)"**

20. **Create Code Node: "Analyze Product Issues"**  
    - Type: Code (JavaScript)  
    - Paste code that parses product issues from Merchant API response, collects disapprovals (critical) and warnings, counts them, and timestamps the check  

21. **Connect "Check Product Status (NEW API)" → "Analyze Product Issues"**

22. **Create IF Node: "IF Disapprovals Found"**  
    - Type: IF  
    - Condition: Numeric, check if `disapproval_count` > 0  

23. **Connect "Analyze Product Issues" → "IF Disapprovals Found"**

24. **Create Slack Node: "Alert - Disapprovals"**  
    - Type: Slack  
    - Configure Slack Webhook for alerts  
    - Message: Include total disapprovals, warnings, list up to 10 critical issues with product title and ID, timestamp, and call to action  

25. **Connect "IF Disapprovals Found" (true branch) → "Alert - Disapprovals"**

26. **Create Slack Node: "Success Summary"**  
    - Type: Slack  
    - Configure Slack Webhook for daily summaries  
    - Message: Summarizes total products processed, quality issues, disapprovals, warnings, optimizations applied, API used, next run time, and timestamp  

27. **Connect "IF Disapprovals Found" (false branch) → "Success Summary"**

28. **Add Sticky Notes** as per workflow positions and contents for documentation and clarity.

29. **Set environment variables and credentials:**  
    - `CHANNABLE_API_URL`, `CHANNABLE_COMPANY_ID`, `CHANNABLE_PROJECT_ID`, `FEED_ID`  
    - `RELEVANCE_AI_API_URL`, `RELEVANCE_TOOL_TITLE_OPTIMIZER_ID`, `RELEVANCE_TOOL_DESCRIPTION_ID`  
    - `MERCHANT_API_URL`, `MERCHANT_ACCOUNT_ID`  
    - HTTP Header Auth credentials for Relevance AI and Channable API  
    - Google API OAuth2 credentials for Merchant API  

30. **Test the workflow manually and verify logs and Slack notifications.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                      | Context or Link                                                                                 |
|----------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Workflow automates Google Shopping feed optimization with AI-powered title/description enhancement and custom label assignment. | Main workflow purpose                                                                          |
| Uses Relevance AI as external AI service for text optimization.                                                                   | External AI integration details                                                                |
| Syncs product data daily before Merchant Center sync to maintain feed quality and minimize disapprovals.                          | Operational best practice                                                                      |
| Uses new Google Merchant API (`merchantapi.googleapis.com`) v2.1 for product upload and status checking, replacing deprecated APIs | Google Merchant API docs: https://developers.google.com/shopping-content                         |
| Slack notifications provide real-time alerts and daily summaries for team awareness.                                              | Slack webhook setup details                                                                    |
| Environment variables keep sensitive info and allow easy configuration changes without modifying workflow nodes.                 | n8n environment variable usage best practices                                                  |

---

This structured documentation fully describes the workflow’s structure, logic, node configuration, and operational flow, enabling maintenance, troubleshooting, and enhancement by advanced users and automated systems.