Transform Shopify Products into SEO Blog Content with DeepSeek AI and Google Sheets

https://n8nworkflows.xyz/workflows/transform-shopify-products-into-seo-blog-content-with-deepseek-ai-and-google-sheets-11394


# Transform Shopify Products into SEO Blog Content with DeepSeek AI and Google Sheets

---

### 1. Workflow Overview

This workflow automates the transformation of Shopify product data into SEO-optimized blog posts using an AI language model and organizes data via Google Sheets. It is designed for e-commerce store owners or marketers who want to leverage existing product information to create engaging, search-engine-friendly blog content without manual writing effort. The workflow consists of the following logical blocks:

- **1.1 Product Data Retrieval:** Fetches product details from Shopify.
- **1.2 Data Backup:** Saves raw product information into Google Sheets for record-keeping.
- **1.3 Processing Loop:** Iterates over each product to generate blog content.
- **1.4 AI Content Generation:** Uses a DeepSeek AI agent to create SEO-friendly blog articles based on product title and description.
- **1.5 AI Output Parsing & Logging:** Parses AI-generated JSON output and stores blog content back to Google Sheets.
- **1.6 Blog Draft Upload:** Uploads generated blog posts as drafts to the Shopify store.
- **1.7 Completion Notification:** Sends an email notification after processing all items.

These blocks are interconnected to ensure reliable, batch-wise processing of products, error handling, data persistence, and notification.

---

### 2. Block-by-Block Analysis

#### 2.1 Product Data Retrieval

- **Overview:**  
  Initiates the workflow by fetching the latest 3 Shopify products using the Shopify API.

- **Nodes Involved:**  
  - When clicking ‘Test workflow’ (Manual Trigger)  
  - Shopify  
  - variables

- **Node Details:**

  - **When clicking ‘Test workflow’**  
    - Type: Manual Trigger  
    - Role: Starts the workflow manually for testing or manual run.  
    - Config: No parameters.  
    - Inputs: None  
    - Outputs: Shopify node  
    - Edge cases: Manual trigger requires user interaction; no input validation needed.

  - **Shopify**  
    - Type: Shopify node  
    - Role: Retrieves product data via Shopify API.  
    - Config:  
      - Resource: Product  
      - Operation: Get All  
      - Limit: 3 products  
      - Authentication: Access Token (configured via credentials)  
    - Inputs: Manual Trigger  
    - Outputs: variables node  
    - Edge cases: API rate limits, authentication errors, empty product list.

  - **variables**  
    - Type: Set Node  
    - Role: Assigns and formats necessary variables for downstream use, including product title, id, description, Shopify shop name, and blog ID.  
    - Config:  
      - Extracts product fields `title`, `id`, `body_html` from input JSON.  
      - Sets static variables: `shop` (shopify store name), `blogId` (target blog for posts).  
    - Inputs: Shopify node  
    - Outputs: Product Data in Google Sheet  
    - Edge cases: Missing or malformed product data; requires valid shop name and blog ID.

---

#### 2.2 Data Backup

- **Overview:**  
  Saves the raw Shopify product data (ID, Title, Description) into a Google Sheet to maintain a record of processed products.

- **Nodes Involved:**  
  - Product Data in Google Sheet

- **Node Details:**

  - **Product Data in Google Sheet**  
    - Type: Google Sheets node  
    - Role: Appends or updates product data in a Google Sheet tab.  
    - Config:  
      - Operation: Append or Update (based on matching ID column).  
      - Columns mapped: ID, Title, Description from variables node.  
      - Document ID and Sheet Name predefined.  
    - Inputs: variables node  
    - Outputs: Loop Over Items  
    - Edge cases: Google API authentication issues, sheet access permissions, column mismatch.

---

#### 2.3 Processing Loop

- **Overview:**  
  Splits the product data into batches (single items) and processes each item sequentially to generate blog content.

- **Nodes Involved:**  
  - Loop Over Items

- **Node Details:**

  - **Loop Over Items**  
    - Type: SplitInBatches  
    - Role: Controls the batch size for processing products one by one.  
    - Config: Default batch size (1 item per batch).  
    - Inputs: Product Data in Google Sheet  
    - Outputs: Wait node (for synchronization), AI Agent  
    - Edge cases: Large batch sizes could overload API calls; batch failure isolation.

---

#### 2.4 AI Content Generation

- **Overview:**  
  Uses DeepSeek AI language model to write SEO-optimized blog posts based on product title and description.

- **Nodes Involved:**  
  - Simple Memory  
  - OpenRouter Chat Model  
  - AI Agent

- **Node Details:**

  - **Simple Memory**  
    - Type: LangChain Memory Buffer Window  
    - Role: Maintains a session-based memory buffer keyed by product ID to preserve context if needed.  
    - Config: Session key based on product ID.  
    - Inputs: None (connected as AI memory for AI Agent)  
    - Outputs: AI Agent (as ai_memory input)  
    - Edge Cases: Memory overflow or session key collisions.

  - **OpenRouter Chat Model**  
    - Type: LangChain OpenRouter Chat Model  
    - Role: Specifies the language model to use (DeepSeek).  
    - Config: Model name `deepseek/deepseek-r1:free`  
    - Inputs: None (connected as language model for AI Agent)  
    - Outputs: AI Agent (as ai_languageModel input)  
    - Edge cases: API limitations, model availability, rate limits.

  - **AI Agent**  
    - Type: LangChain AI Agent  
    - Role: Main logic node that sends the product info prompt to DeepSeek and expects JSON output.  
    - Config:  
      - Prompt includes instructions to generate SEO-friendly blog title and HTML content from product data.  
      - Uses product title and description from `variables` node via expressions.  
      - Output parser expects JSON with keys `blogTitle` and `blogContent`.  
    - Inputs: Loop Over Items (main), Simple Memory (ai_memory), OpenRouter Chat Model (ai_languageModel)  
    - Outputs: Code node  
    - Edge cases: AI may fail to return valid JSON; timeout; prompt interpretation errors.

---

#### 2.5 AI Output Parsing & Logging

- **Overview:**  
  Parses the AI-generated JSON string, extracts blog title and content, handles errors, and saves the result into Google Sheets.

- **Nodes Involved:**  
  - Code  
  - Google Sheets1

- **Node Details:**

  - **Code**  
    - Type: Code (JavaScript)  
    - Role: Parses the AI output string to extract JSON object reliably.  
    - Config:  
      - Uses regex to find JSON in AI output.  
      - Extracts `blogTitle` or fallback `title`, and `blogContent` or fallback `description`.  
      - Throws error if parsing fails or keys missing.  
      - Logs error and returns error info on failure.  
    - Inputs: AI Agent  
    - Outputs: Google Sheets1  
    - Edge cases: Malformed AI output, missing keys, JSON parse errors.

  - **Google Sheets1**  
    - Type: Google Sheets node  
    - Role: Appends or updates blog title and content into a Google Sheet tab named "Blogs".  
    - Config:  
      - Columns mapped: Title (`blogTitle`), Description (`blogContent`).  
      - Matches on Title column.  
      - Document ID and Sheet Name predefined.  
    - Inputs: Code node  
    - Outputs: Create Blog As Draft  
    - Edge cases: Google Sheets API errors, sheet access, duplicate titles.

---

#### 2.6 Blog Draft Upload

- **Overview:**  
  Uploads the generated blog post content as a draft article into the Shopify blog specified by Blog ID.

- **Nodes Involved:**  
  - Create Blog As Draft

- **Node Details:**

  - **Create Blog As Draft**  
    - Type: HTTP Request  
    - Role: Calls Shopify Admin API to create a draft blog article.  
    - Config:  
      - POST to URL constructed dynamically with shop name and blog ID from variables.  
      - JSON body includes article title, HTML content, tags, and published status set to false (draft).  
      - Authentication via Shopify Access Token credentials.  
    - Inputs: Google Sheets1  
    - Outputs: Loop Over Items (to continue processing)  
    - Edge cases: API authentication failure, invalid blog ID, request rate limits, malformed content.

---

#### 2.7 Completion Notification

- **Overview:**  
  After all products are processed, sends an email notification confirming blog drafts are created.

- **Nodes Involved:**  
  - Wait  
  - Gmail

- **Node Details:**

  - **Wait**  
    - Type: Wait node  
    - Role: Pauses execution, ensuring all item processing completes before notification.  
    - Config: Waits 1 second, executes once.  
    - Inputs: Loop Over Items  
    - Outputs: Gmail node  
    - Edge cases: Insufficient wait time if processing exceeds expected duration.

  - **Gmail**  
    - Type: Gmail node  
    - Role: Sends email notification to configured recipient.  
    - Config:  
      - Recipient: `youremail@gmail.com` (placeholder)  
      - Subject: "Blog Posts Are Created"  
      - Message: Confirmation message stating blog drafts created.  
      - Uses Gmail OAuth2 credentials.  
    - Inputs: Wait node  
    - Outputs: None (end of workflow)  
    - Edge cases: Email sending failure, invalid recipient address, Gmail API quota.

---

### 3. Summary Table

| Node Name                 | Node Type                        | Functional Role                        | Input Node(s)           | Output Node(s)                | Sticky Note                                                                                                  |
|---------------------------|---------------------------------|-------------------------------------|-------------------------|------------------------------|--------------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger                  | Starts workflow manually              | None                    | Shopify                      |                                                                                                              |
| Shopify                   | Shopify API Node                 | Fetch latest Shopify products        | When clicking ‘Test workflow’ | variables                   | ## Fetch Products  Starts the workflow and fetches the latest product list from your Shopify store.          |
| variables                 | Set Node                        | Assigns product and workflow variables | Shopify                  | Product Data in Google Sheet  | ## Data Backup  Sets necessary variables and saves the raw product data into Google Sheets for record-keeping. |
| Product Data in Google Sheet | Google Sheets                   | Logs raw product data                 | variables                | Loop Over Items              |                                                                                                              |
| Loop Over Items           | SplitInBatches                  | Processes each product individually  | Product Data in Google Sheet | Wait, AI Agent               | ## AI Generation Loop  Iterates through each product. The AI Agent writes an SEO-friendly blog post for each item. |
| Wait                      | Wait Node                      | Waits for processing completion      | Loop Over Items          | Gmail                        | ## Completion Notify  Waits for the loop to finish processing all items, then sends an email confirmation.   |
| Gmail                     | Gmail Node                     | Sends completion email notification  | Wait                     | None                        |                                                                                                              |
| AI Agent                  | LangChain AI Agent             | Generates SEO blog content using AI | Loop Over Items          | Code                        | ## AI Model Config  Defines the LLM (DeepSeek) and memory settings used by the AI Agent.                      |
| Simple Memory             | LangChain Memory Buffer Window | Maintains AI session memory          | None (ai_memory input)   | AI Agent (ai_memory input)   | ## AI Model Config  Defines the LLM (DeepSeek) and memory settings used by the AI Agent.                      |
| OpenRouter Chat Model     | LangChain OpenRouter Chat Model | Specifies DeepSeek LLM for AI Agent  | None (ai_languageModel input) | AI Agent (ai_languageModel input) | ## AI Model Config  Defines the LLM (DeepSeek) and memory settings used by the AI Agent.                      |
| Code                      | Code Node                     | Parses AI JSON output and error handling | AI Agent                | Google Sheets1               | ## Parse & Log  Cleans the AI output using code and saves the final blog title and content to Google Sheets.  |
| Google Sheets1            | Google Sheets                  | Logs generated blog posts             | Code                     | Create Blog As Draft         |                                                                                                              |
| Create Blog As Draft      | HTTP Request (Shopify API)      | Uploads blog post draft to Shopify   | Google Sheets1            | Loop Over Items              | ## Upload Draft  Uploads the generated content to Shopify as a draft blog post, ready for review.             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node:**  
   - Type: Manual Trigger  
   - Purpose: To start the workflow manually.

2. **Add Shopify Node:**  
   - Type: Shopify  
   - Operation: Get All Products  
   - Limit: 3  
   - Authentication: Shopify Access Token credentials configured with your store’s API access.  
   - Connect Manual Trigger output to Shopify input.

3. **Add Set Node (Variables):**  
   - Type: Set  
   - Assign variables from Shopify product data:  
     - `title` = `{{$json["title"]}}`  
     - `id` = `{{$json["id"]}}`  
     - `body_html` = `{{$json["body_html"]}}`  
     - `shop` = Your Shopify shop name (e.g., "developmentwebsensepro")  
     - `blogId` = Target Shopify blog ID (e.g., "94126407876")  
   - Connect Shopify output to Variables input.

4. **Add Google Sheets Node (Product Data Backup):**  
   - Type: Google Sheets  
   - Operation: Append or Update  
   - Document ID: Your Google Sheet ID for raw product data  
   - Sheet Name or GID: "Sheet1" or appropriate tab  
   - Columns to map:  
     - ID = `{{$json["id"]}}`  
     - Title = `{{$json["title"]}}`  
     - Description = `{{$json["body_html"]}}`  
   - Matching column: ID  
   - Connect Variables output to this node.

5. **Add SplitInBatches Node:**  
   - Type: SplitInBatches  
   - Batch size: default (1)  
   - Connect Google Sheets output to SplitInBatches input.

6. **Add LangChain Memory Node (Simple Memory):**  
   - Type: LangChain Memory Buffer Window  
   - Session Key: `{{$json["id"]}}` (to isolate memory per product)  
   - No input connection (will be linked to AI Agent as memory input).

7. **Add LangChain OpenRouter Chat Model Node:**  
   - Type: LangChain OpenRouter Chat Model  
   - Model: `deepseek/deepseek-r1:free`  
   - No input connection (will be linked to AI Agent as language model input).

8. **Add LangChain AI Agent Node:**  
   - Type: LangChain AI Agent  
   - Prompt: Provide detailed instructions to generate SEO blog post with keys `blogTitle` and `blogContent`.  
   - Use expressions to insert product title and description from `variables` node.  
   - Link SplitInBatches main output as input.  
   - Link Simple Memory as `ai_memory` input.  
   - Link OpenRouter Chat Model as `ai_languageModel` input.

9. **Add Code Node:**  
   - Type: Code (JavaScript)  
   - Purpose: Extract JSON object from AI output, parse, validate, and return normalized blogTitle and blogContent keys.  
   - Connect AI Agent main output to Code input.

10. **Add Google Sheets Node (Blog Content Logging):**  
    - Type: Google Sheets  
    - Operation: Append or Update  
    - Document ID: Same Google Sheet ID or a different one for blogs  
    - Sheet Name or GID: "Blogs"  
    - Columns:  
      - Title = `{{$json["blogTitle"]}}`  
      - Description = `{{$json["blogContent"]}}`  
    - Matching column: Title  
    - Connect Code output to this node.

11. **Add HTTP Request Node (Create Blog As Draft):**  
    - Type: HTTP Request  
    - Method: POST  
    - URL: `https://{{ $json["shop"] }}.myshopify.com/admin/api/2024-01/blogs/{{ $json["blogId"] }}/articles.json`  
    - Authentication: Shopify Access Token credentials  
    - Body (JSON):  
      ```json
      {
        "article": {
          "title": "{{ $json['blogTitle'] }}",
          "body_html": "{{ $json['blogContent'] }}",
          "tags": "nutrition, wellness, product, shopify",
          "published": false
        }
      }
      ```  
    - Connect Google Sheets1 output to this node.

12. **Connect HTTP Request output back to SplitInBatches input:**  
    - Enables iterative processing loop.

13. **Add Wait Node:**  
    - Type: Wait  
    - Amount: 1 second  
    - Execute Once: true  
    - Connect SplitInBatches output (main) to Wait input.

14. **Add Gmail Node:**  
    - Type: Gmail  
    - Configuration: OAuth2 with your Gmail account  
    - To: Your email address  
    - Subject: "Blog Posts Are Created"  
    - Message: "I am your AI Agent: Good news, I have added all the blog posts as draft after reading all your product data."  
    - Connect Wait output to Gmail input.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                | Context or Link                                                          |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------|
| Free AI Agent by WebSensePro — Complete free AI agent to read Shopify products and create blogs based on product Title and Description.                     | Included as sticky note in workflow.                                    |
| Setup steps include filling Shopify shop name and blog ID inside Variables node, connecting accounts for Shopify, Google Sheets, OpenRouter, and Gmail.     | Workflow setup instructions.                                            |
| Video tutorial walkthrough available: https://www.youtube.com/watch?v=RCwFc57V6oo                                                                           | Linked in sticky note at workflow start.                               |
| Error handling implemented in Code node to handle malformed AI JSON output robustly.                                                                         | Key to maintain workflow stability.                                    |
| The workflow processes each product individually in batches of one to ensure isolated error handling and easier reprocessing.                              | Design choice for reliability.                                          |

---

**Disclaimer:**  
The provided text is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.

---