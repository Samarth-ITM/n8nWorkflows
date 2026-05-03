Synchronizing WooCommerce Inventory and Creating Products with Google Gemini AI and BrowserAct

https://n8nworkflows.xyz/workflows/synchronizing-woocommerce-inventory-and-creating-products-with-google-gemini-ai-and-browseract-10089


# Synchronizing WooCommerce Inventory and Creating Products with Google Gemini AI and BrowserAct

### 1. Workflow Overview

This workflow automates the synchronization of WooCommerce product inventory and the intelligent creation of new products using Google Gemini AI and BrowserAct scraping. It is designed to manage WooCommerce product data dynamically by scraping supplier inventory pages, updating existing product prices and stock, and creating new products with AI-generated descriptions based on detailed scraped attributes.

Logical blocks in this workflow:

- **1.1 Input Reception and Supplier Loop**: Reads a list of suppliers and their inventory URLs from Google Sheets and iterates over each supplier.
- **1.2 Inventory Scraping and Product Extraction**: Uses BrowserAct to scrape product stock data from supplier pages, then parses the scraped JSON into individual product items.
- **1.3 Product Processing Loop**: Iterates over each scraped product, searching WooCommerce to determine if it exists.
- **1.4 Existing Product Update**: If the product exists, updates price and stock quantity in WooCommerce.
- **1.5 New Product Creation**: If the product does not exist, optionally filters by category, scrapes detailed product attributes via BrowserAct, uses Google Gemini AI to format these attributes into an HTML product description, and creates the new product in WooCommerce.
- **1.6 Error Handling and Notifications**: Sends Slack notifications for errors occurring at various points such as scraping or WooCommerce updates/creations and signals completion.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Supplier Loop

- **Overview:**  
  Reads supplier data from a Google Sheet and loops through each supplier to start inventory scraping.

- **Nodes Involved:**  
  - When clicking ‘Execute workflow’ (Manual Trigger)  
  - Get Suppliers & Links (Google Sheets)  
  - Loop Over Suppliers (Split In Batches)  

- **Node Details:**

  - **When clicking ‘Execute workflow’**  
    - Type: Manual Trigger  
    - Role: Starts workflow execution manually.  
    - Connections: Outputs to "Get Suppliers & Links".  
    - Potential Failures: None.

  - **Get Suppliers & Links**  
    - Type: Google Sheets  
    - Role: Fetches supplier list including inventory URLs and product types from the sheet named "gid=0" in a specific Google Sheets document.  
    - Credentials: Google Sheets OAuth2 API.  
    - Important Parameters: Document ID and Sheet Name set to the correct spreadsheet and tab.  
    - Connections: Outputs to "Loop Over Suppliers".  
    - Edge Cases: Empty or malformed sheet data could disrupt the loop.

  - **Loop Over Suppliers**  
    - Type: Split In Batches  
    - Role: Processes each supplier from the sheet individually to control flow and resource usage.  
    - Connections:  
      - Main output 0: to "Send a message" (Slack completion notice)  
      - Main output 1: to "Get Product Stock Data" (BrowserAct scraping)  
    - Edge Cases: Large supplier lists may require batching tuning.

---

#### 1.2 Inventory Scraping and Product Extraction

- **Overview:**  
  Scrapes the supplier’s inventory page using BrowserAct, then parses the scraped JSON string into individual product items.

- **Nodes Involved:**  
  - Get Product Stock Data (BrowserAct)  
  - BrowserAct JSON Breakdown (Code)  
  - Send BrowserAct Error (Slack)  

- **Node Details:**

  - **Get Product Stock Data**  
    - Type: BrowserAct workflow execution  
    - Role: Runs BrowserAct scraper workflow (ID: 58839712632662919) on the supplier’s inventory URL to get current stock data.  
    - Input: Takes supplier page URL from the supplier item JSON.  
    - Credentials: BrowserAct API.  
    - Parameters: Maximum wait time 900 seconds, polling interval 30 seconds, saves browser data.  
    - Output: Scraped JSON string of product stock data.  
    - On Error: Continues and routes to Slack error notification.  
    - Potential Errors: Timeout, network issues, scraper failure.

  - **BrowserAct JSON Breakdown**  
    - Type: Code Node (JavaScript)  
    - Role: Parses the JSON string from BrowserAct output into multiple n8n items, one per product.  
    - Key Logic:  
      - Validates presence of JSON string.  
      - Parses JSON, throws error if malformed or not an array.  
      - Outputs parsed array as separate items.  
    - Input: Output of BrowserAct node (JSON string path: `$input.first().json.output.string`).  
    - Output: Multiple product items.  
    - Edge Cases: Empty or malformed JSON string causes workflow error.

  - **Send BrowserAct Error**  
    - Type: Slack  
    - Role: Sends Slack alert if BrowserAct scraping fails.  
    - Credentials: Slack OAuth2.  
    - Parameters: Posts to a configured Slack channel (ID: C09LWT82KHN).  
    - Triggered on BrowserAct node error.

---

#### 1.3 Product Processing Loop

- **Overview:**  
  Loops over each scraped product, checks if it exists in WooCommerce by name, and routes to update or creation logic based on existence.

- **Nodes Involved:**  
  - Loop Over Products (Split In Batches)  
  - Get many products (WooCommerce)  
  - Check for Product availability (If)  

- **Node Details:**

  - **Loop Over Products**  
    - Type: Split In Batches  
    - Role: Processes each product individually to manage resource limits.  
    - Connections:  
      - Output 0: To "Loop Over Suppliers" (batch control)  
      - Output 1: To "Get many products".  

  - **Get many products**  
    - Type: WooCommerce  
    - Role: Searches WooCommerce store for products matching the scraped product name (`search` parameter set to product’s `Name`).  
    - Credentials: WooCommerce API.  
    - Output: List of matching products (could be empty).  
    - Edge Cases: API errors, rate limits.

  - **Check for Product availability**  
    - Type: If  
    - Role: Checks if WooCommerce returned any matching product for the current scraped item.  
    - Logic: Routes true if product exists, false if not.  
    - Outputs:  
      - True: To "Update a product" node.  
      - False: To "Check the Category" node for new product creation filtering.

---

#### 1.4 Existing Product Update

- **Overview:**  
  Updates existing WooCommerce product price and stock quantity using the scraped data.

- **Nodes Involved:**  
  - Update a product (WooCommerce)  
  - Send Error (Slack)  

- **Node Details:**

  - **Update a product**  
    - Type: WooCommerce  
    - Role: Updates product identified by `id` with new `regularPrice` and `stockQuantity` from parsed BrowserAct JSON data (`Price` and `Quantity` fields).  
    - Credentials: WooCommerce API.  
    - Parameters: Continues on error to prevent workflow halt.  
    - Connections:  
      - On success: To "Merge" node (combining update and creation flows).  
      - On error: To "Send Error" Slack node.  
    - Edge Cases: WooCommerce API errors, invalid product IDs, data conversion errors.

  - **Send Error**  
    - Type: Slack  
    - Role: Sends Slack notification on WooCommerce update or creation errors.  
    - Credentials: Slack OAuth2.  
    - Parameters: Posts a generic error message to configured Slack channel.

---

#### 1.5 New Product Creation

- **Overview:**  
  For missing products, optionally filters by category, scrapes detailed attributes from a secondary source, uses Google Gemini AI to generate an HTML description, and creates the new product in WooCommerce.

- **Nodes Involved:**  
  - Check the Category (If)  
  - Add Missing Product (BrowserAct)  
  - BrowserAct Json Breakdown (Code)  
  - Google Gemini Chat Model (AI Language Model)  
  - Create the HTML Data Table (LangChain AI Agent)  
  - Create a product (WooCommerce)  
  - Send Error 2 (Slack)  

- **Node Details:**

  - **Check the Category**  
    - Type: If  
    - Role: Optional filter to allow new product creation only if the scraped product’s category matches the supplier’s specified product type.  
    - Logic: Compares product category with supplier’s product type field, case-insensitive.  
    - Outputs:  
      - True: To "Add Missing Product" (scraping detailed attributes).  
      - False: Loops back to "Loop Over Products" (skips creation).  

  - **Add Missing Product**  
    - Type: BrowserAct workflow execution  
    - Role: Runs secondary BrowserAct scraper (ID: 58839211174254215) to fetch detailed product attributes from external source (e.g., DigiKey).  
    - Input Parameters: Product name and a constant supplier URL ("https://www.digikey.co.uk/en/").  
    - Credentials: BrowserAct API.  
    - On Error: Continues and triggers Slack "Send Error 2".  
    - Output: JSON string of detailed product attributes.  

  - **BrowserAct Json Breakdown**  
    - Type: Code Node (JavaScript)  
    - Role: Parses JSON string output from the secondary BrowserAct scraper into multiple items (typically one).  
    - Logic: Same robust parsing as first BrowserAct JSON Breakdown node.  
    - Output: Parsed attribute object.  

  - **Google Gemini Chat Model**  
    - Type: AI Language Model (Google Gemini)  
    - Role: Processes scraped product attributes to structure a response for the AI Agent.  
    - Credentials: Google Gemini (PaLM) API.  
    - Connection: Outputs to "Create the HTML Data Table".

  - **Create the HTML Data Table**  
    - Type: LangChain AI Agent  
    - Role: Converts product attribute text into a precise HTML table with specific tags, classes, and formatting rules as per instructions.  
    - Input: Text of product attributes from Gemini output.  
    - Output: HTML string representing product description.  
    - Important: Only outputs final HTML code without extra formatting or markdown.  

  - **Create a product**  
    - Type: WooCommerce  
    - Role: Creates new WooCommerce product with scraped name, SKU, price, stock, and AI-generated HTML description.  
    - Parameters:  
      - SKU generated as "SKU_" plus product name.  
      - Stock management enabled.  
    - Credentials: WooCommerce API.  
    - On Error: Continues and triggers "Send Error".  
    - Output: To "Merge" node.

  - **Send Error 2**  
    - Type: Slack  
    - Role: Sends Slack alerts for errors occurring in the secondary BrowserAct node.  
    - Credentials: Slack OAuth2.  
    - Posts to configured Slack channel.

---

#### 1.6 Merge and Completion Notifications

- **Overview:**  
  Merges the flows from product update and creation paths and sends a final Slack message upon completion.

- **Nodes Involved:**  
  - Merge (Merge node)  
  - Send a message (Slack)  

- **Node Details:**

  - **Merge**  
    - Type: Merge  
    - Role: Combines outputs from product updates, new product creation, and error notifications into a single flow to finalize the process.  
    - Inputs: 3 inputs (update success, create success, error notifications).  
    - Output: To "Send a message" Slack node.  

  - **Send a message**  
    - Type: Slack  
    - Role: Sends notification to Slack channel indicating the process has finished.  
    - Credentials: Slack OAuth2.  
    - Parameters: Channel ID set to notify the team (C09LWT82KHN).  

---

### 3. Summary Table

| Node Name                  | Node Type                                | Functional Role                                         | Input Node(s)                  | Output Node(s)                    | Sticky Note                                                                                                                               |
|----------------------------|-----------------------------------------|---------------------------------------------------------|-------------------------------|---------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger                      | Starts the workflow manually                             | —                             | Get Suppliers & Links             | See "Sticky Note - Intro" for workflow overview and usage instructions                                                                    |
| Get Suppliers & Links       | Google Sheets                           | Reads supplier inventory URLs and product types         | When clicking ‘Execute workflow’ | Loop Over Suppliers             | See "Sticky Note - Intro" and "Sticky Note - Input & Scrape Loop"                                                                         |
| Loop Over Suppliers         | Split In Batches                        | Loops over supplier list to batch process                | Get Suppliers & Links           | Send a message, Get Product Stock Data | See "Sticky Note - Input & Scrape Loop"                                                                                                |
| Get Product Stock Data      | BrowserAct Workflow                     | Scrapes supplier inventory page for product stock data  | Loop Over Suppliers             | BrowserAct JSON Breakdown, Send BrowserAct Error | See "Sticky Note - Input & Scrape Loop"                                                                                                |
| BrowserAct JSON Breakdown   | Code                                   | Parses scraped JSON string into individual product items | Get Product Stock Data          | Loop Over Products               | See "Sticky Note - Input & Scrape Loop"                                                                                                |
| Send BrowserAct Error       | Slack                                  | Sends Slack alert on first BrowserAct scraping failure  | Get Product Stock Data (on error) | —                            | See "Sticky Note - Input & Scrape Loop"                                                                                                |
| Loop Over Products          | Split In Batches                        | Loops over each scraped product                          | BrowserAct JSON Breakdown       | Loop Over Suppliers, Get many products | See "Sticky Note - Product Processing Loop"                                                                                             |
| Get many products           | WooCommerce                            | Searches WooCommerce for product existence by name      | Loop Over Products              | Check for Product availability   | See "Sticky Note - Product Processing Loop"                                                                                             |
| Check for Product availability | If                                  | Determines if product exists in WooCommerce              | Get many products              | Update a product, Check the Category | See "Sticky Note - Product Processing Loop"                                                                                             |
| Update a product            | WooCommerce                            | Updates existing product price and stock                 | Check for Product availability | Merge, Send Error                | See "Sticky Note - Product Processing Loop"                                                                                             |
| Send Error                 | Slack                                  | Sends error notification on update/create failure       | Update a product, Create a product (on error) | Merge                         | See "Sticky Note - Create New Product1"                                                                                                |
| Check the Category          | If                                     | Filters missing products by category for creation        | Check for Product availability | Add Missing Product, Loop Over Products | See "Sticky Note - Create New Product"                                                                                                |
| Add Missing Product         | BrowserAct Workflow                     | Scrapes detailed product attributes from secondary source | Check the Category              | BrowserAct Json Breakdown, Send Error 2 | See "Sticky Note - Create New Product"                                                                                                |
| BrowserAct Json Breakdown   | Code                                   | Parses detailed attribute JSON string                    | Add Missing Product             | Create the HTML Data Table       | See "Sticky Note - Create New Product"                                                                                                |
| Send Error 2                | Slack                                  | Sends alert on error in secondary BrowserAct scraping   | Add Missing Product (on error)  | —                              | See "Sticky Note - Create New Product"                                                                                                |
| Google Gemini Chat Model    | AI Language Model (Google Gemini)       | Processes product attributes text for AI agent          | BrowserAct Json Breakdown       | Create the HTML Data Table       | See "Sticky Note - Create New Product"                                                                                                |
| Create the HTML Data Table  | LangChain AI Agent                      | Converts attributes text into HTML table for description | Google Gemini Chat Model        | Create a product                 | See "Sticky Note - Create New Product"                                                                                                |
| Create a product            | WooCommerce                            | Creates new product with AI-generated description        | Create the HTML Data Table      | Merge, Send Error                | See "Sticky Note - Create New Product"                                                                                                |
| Merge                      | Merge                                  | Combines update, creation, and error notification flows | Update a product, Create a product, Send Error | Send a message                | —                                                                                                                                       |
| Send a message              | Slack                                  | Sends final notification that syncing process finished  | Loop Over Suppliers, Merge      | —                               | —                                                                                                                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger node**  
   - Type: Manual Trigger  
   - No parameters.  
   - This will start the workflow manually.

2. **Add Google Sheets node "Get Suppliers & Links"**  
   - Credentials: Google Sheets OAuth2 account.  
   - Parameters:  
     - Document ID: The ID of your supplier list spreadsheet.  
     - Sheet Name: The tab containing supplier data (e.g., gid=0).  
   - Connect from Manual Trigger node.

3. **Add Split In Batches node "Loop Over Suppliers"**  
   - No special parameters by default.  
   - Connect from "Get Suppliers & Links".

4. **Add BrowserAct node "Get Product Stock Data"**  
   - Credentials: BrowserAct API account.  
   - Parameters:  
     - Workflow ID: Use your BrowserAct workflow for scraping supplier inventory (e.g., "58839712632662919").  
     - Input parameter: Pass the supplier’s inventory page URL (`Target_Url` = `{{$json["Inventory page"]}}`).  
     - Set max wait time 900 seconds and polling interval 30 seconds.  
     - Enable saving browser data.  
   - Connect from "Loop Over Suppliers" (output index 1).

5. **Add Code node "BrowserAct JSON Breakdown"**  
   - Paste JavaScript to parse `output.string` JSON into array items.  
   - Connect from "Get Product Stock Data".

6. **Add Slack node "Send BrowserAct Error"**  
   - Credentials: Slack OAuth2.  
   - Parameters: Set channel ID for notifications.  
   - Connect as error output from "Get Product Stock Data".

7. **Add Split In Batches node "Loop Over Products"**  
   - No special parameters.  
   - Connect from "BrowserAct JSON Breakdown".

8. **Add WooCommerce node "Get many products"**  
   - Credentials: WooCommerce API.  
   - Parameters:  
     - Operation: getAll.  
     - Search: Set to product name field `={{ $json.Name }}`.  
   - Connect from "Loop Over Products".

9. **Add If node "Check for Product availability"**  
   - Condition: Check if the WooCommerce search returned any product (non-empty).  
   - Connect from "Get many products".  

10. **Add WooCommerce node "Update a product"**  
    - Credentials: WooCommerce API.  
    - Parameters:  
      - Operation: update.  
      - Product ID: `={{ $json.id }}` (from WooCommerce search).  
      - Update Fields:  
        - regularPrice: `={{ $('BrowserAct JSON Breakdown').item.json.Price }}`  
        - stockQuantity: `={{ $('BrowserAct JSON Breakdown').item.json.Quantity }}`  
      - Set "Continue on error" enabled.  
    - Connect True output from "Check for Product availability".

11. **Add Slack node "Send Error"**  
    - Credentials: Slack OAuth2.  
    - Parameters: Error message for update/create failures.  
    - Connect error outputs of "Update a product" and "Create a product" to this node.

12. **Add If node "Check the Category"**  
    - Condition: Compare scraped product category to supplier's product type (case-insensitive equality).  
    - Connect False output from "Check for Product availability".

13. **Add BrowserAct node "Add Missing Product"**  
    - Credentials: BrowserAct API.  
    - Parameters:  
      - Workflow ID: Your second BrowserAct workflow for detailed product attributes (e.g., "58839211174254215").  
      - Input parameters: Product_Name (`={{ $('BrowserAct JSON Breakdown').item.json.Name }}`) and Supplier_Link ("https://www.digikey.co.uk/en/").  
      - Max wait time: 900 seconds.  
      - Polling interval: 30 seconds.  
    - Connect True output from "Check the Category".  
    - Enable "Continue on error".

14. **Add Code node "BrowserAct Json Breakdown"**  
    - Same parsing script as for first BrowserAct JSON breakdown.  
    - Connect from "Add Missing Product".

15. **Add Slack node "Send Error 2"**  
    - Credentials: Slack OAuth2.  
    - Parameters: Error message for second BrowserAct failure.  
    - Connect error output from "Add Missing Product".

16. **Add Google Gemini Chat Model node**  
    - Credentials: Google Gemini (PaLM) API account.  
    - Parameters: Default or as per your AI usage.  
    - Connect from "BrowserAct Json Breakdown".

17. **Add LangChain AI Agent node "Create the HTML Data Table"**  
    - Parameters: Set system prompt and instructions to transform product attributes into HTML table (use detailed prompt from the workflow).  
    - Connect AI output from "Google Gemini Chat Model".

18. **Add WooCommerce node "Create a product"**  
    - Credentials: WooCommerce API.  
    - Parameters:  
      - Operation: create product.  
      - Name: `={{ $('BrowserAct Json Breakdown').item.json.Name }}`  
      - SKU: `=SKU_{{ $('BrowserAct Json Breakdown').item.json.Name }}`  
      - Description: AI-generated HTML from previous node.  
      - Manage stock: true  
      - regularPrice: `={{ $('BrowserAct Json Breakdown').item.json.Price }}`  
      - stockQuantity: `={{ $('BrowserAct Json Breakdown').item.json.Quantity }}`  
      - Continue on error enabled.  
    - Connect from "Create the HTML Data Table".

19. **Add Merge node "Merge"**  
    - Parameters: Number of inputs = 3.  
    - Connect outputs:  
      - From "Update a product" (success)  
      - From "Create a product" (success)  
      - From "Send Error" (error notifications)  

20. **Add Slack node "Send a message"**  
    - Credentials: Slack OAuth2.  
    - Parameters: Message "The product syncing process finished."  
    - Connect from "Merge".

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                             | Context or Link                                                                                   |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| This workflow requires BrowserAct API account and two BrowserAct templates named “WooCommerce Inventory & Stock Synchronization” and “WooCommerce Product Data Reconciliation”.                                                                                           | BrowserAct nodes package: https://www.npmjs.com/package/n8n-nodes-browseract-workflows           |
| Slack notifications are used throughout the workflow to alert on errors and completion status. Update Slack channel IDs as appropriate.                                                                                                                                | Slack OAuth2 credentials and channel configuration                                              |
| Google Gemini (PaLM) is used as the AI language model to generate structured HTML descriptions from product attributes. Credentials must be configured.                                                                                                                 | Google Gemini API account                                                                        |
| For detailed instructions and troubleshooting, join the BrowserAct Discord community or visit the BrowserAct blog.                                                                                                                                                      | Discord: https://discord.com/invite/UpnCKd7GaU, Blog: https://www.browseract.com/blog           |
| Videos explaining setup and usage of BrowserAct API keys, workflows, and customization are available publicly.                                                                                                                                                          | YouTube playlist linked in sticky notes                                                          |
| The workflow includes robust error handling, with Slack messages sent whenever scraping or WooCommerce operations fail, allowing monitoring and quick response.                                                                                                         | See Slack error nodes and workflow logic                                                        |
| SKU generation is simplistic and based on product name prefixed with “SKU_”; adjust as needed for your SKU conventions.                                                                                                                                                 | SKU format: `SKU_{{product name}}`                                                              |
| The AI agent for HTML table generation strictly follows a detailed prompt specifying exact HTML structure, classes, and formatting for product descriptions ensuring consistent front-end display.                                                                       | See "Create the HTML Data Table" node prompt                                                    |

---

**Disclaimer:** The text provided is exclusively from an automated workflow created with n8n integration and automation tools. It strictly complies with current content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.