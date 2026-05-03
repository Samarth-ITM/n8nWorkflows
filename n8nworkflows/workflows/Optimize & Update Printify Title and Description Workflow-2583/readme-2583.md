Optimize & Update Printify Title and Description Workflow

https://n8nworkflows.xyz/workflows/optimize---update-printify-title-and-description-workflow-2583


# Optimize & Update Printify Title and Description Workflow

### 1. Workflow Overview

This n8n workflow automates the retrieval, optimization, and update of product titles and descriptions in Printify shops. It targets ecommerce businesses that want to enhance product listings by leveraging AI-generated content, maintaining brand consistency, and managing updates efficiently.

**Target Use Cases:**
- Automating product content optimization for Printify stores.
- Generating engaging, SEO-friendly product titles and descriptions using OpenAI's GPT model.
- Tracking product updates and statuses via Google Sheets.
- Supporting batch processing for large catalogs.
- Customizing output tone and seasonal instructions per brand guidelines.

**Logical Blocks:**

- **1.1 Input Reception & Brand Setup:** Manual trigger to start workflow and initialization of brand-specific guidelines.
- **1.2 Printify Data Retrieval:** Fetch shops and their products from Printify API.
- **1.3 Product Iteration & Batch Processing:** Splitting product lists into individual items and processing them in batches.
- **1.4 AI Content Generation:** Using OpenAI to generate optimized product titles and descriptions with brand tone and custom instructions.
- **1.5 Google Sheets Logging & Monitoring:** Logging product information and changes; triggering workflow upon Google Sheets row updates.
- **1.6 Conditional Logic & Update Control:** Decision points to control processing flow based on conditions like update readiness.
- **1.7 Printify Product Update:** Sending updated product data back to Printify.
- **1.8 Utility & Support Nodes:** Includes calculation tools and informational sticky notes.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Brand Setup

**Overview:**  
Initial entry point for manual execution; sets brand guidelines including brand name, tone, and seasonal instructions that influence AI content generation.

**Nodes Involved:**  
- When clicking ‘Test workflow’ (Manual Trigger)  
- Brand Guidelines + Custom Instructions (Set)  
- Sticky Note11 (Sticky Note)

**Node Details:**  
- **When clicking ‘Test workflow’**  
  - Type: Manual Trigger  
  - Config: No parameters; used for testing or manual start.  
  - Outputs: Triggers "Brand Guidelines + Custom Instructions"  
  - Potential issues: None; manual start only.

- **Brand Guidelines + Custom Instructions**  
  - Type: Set Node  
  - Config: Sets three string variables — `brand_name` ("AlexK1919"), `brand_tone` ("informal, instructional, trustoworthy"), and `custom_instructions` ("re-write for the coming Christmas season").  
  - Inputs: Manual Trigger  
  - Outputs: To "Printify - Get Shops"  
  - Edge cases: User must update brand info before running; incorrect values could affect AI output tone.

- **Sticky Note11**  
  - Role: Reminds users to update brand guidelines before running workflow.

---

#### 1.2 Printify Data Retrieval

**Overview:**  
Calls Printify API to retrieve the list of shops and then fetches products for each shop.

**Nodes Involved:**  
- Printify - Get Shops (HTTP Request)  
- Printify - Get Products (HTTP Request)  
- Split Out (SplitOut)  
- Sticky Note10 (Sticky Note)

**Node Details:**  
- **Printify - Get Shops**  
  - Type: HTTP Request  
  - Config: GET to `https://api.printify.com/v1/shops.json` with HTTP Header Auth (Printify API Key).  
  - Inputs: From Brand Guidelines node  
  - Outputs: Shop list JSON to Printify - Get Products  
  - Failure modes: Auth errors if API key invalid; network timeout.

- **Printify - Get Products**  
  - Type: HTTP Request  
  - Config: GET to `https://api.printify.com/v1/shops/{{ $json.id }}/products.json` dynamically per shop ID.  
  - Inputs: Shops list  
  - Outputs: Product data to Split Out node  
  - Failure modes: Auth errors, shop ID missing, API rate limits.

- **Split Out**  
  - Type: SplitOut  
  - Config: Splits "data" field array to individual product records.  
  - Inputs: Product list  
  - Outputs: Single product items to Loop Over Items  
  - Edge cases: Empty product lists result in no downstream processing.

- **Sticky Note10**  
  - Content: Notes that API calls can be swapped for similar services (Printful, Vistaprint).

---

#### 1.3 Product Iteration & Batch Processing

**Overview:**  
Processes each product individually or in batches to manage workflow load efficiently.

**Nodes Involved:**  
- Loop Over Items (SplitInBatches)  
- Split - id, title, desc (SplitOut)  
- Number of Options (Set)  
- Calculate Options (Code)  
- If1 (If)  
- Remember Options (Set)  
- Sticky Note1, Sticky Note2 (Sticky Notes)

**Node Details:**  
- **Loop Over Items**  
  - Type: SplitInBatches  
  - Config: Default batch size (can be configured manually).  
  - Inputs: Individual products from Split Out  
  - Outputs: To Split - id, title, desc for further processing  
  - Edge cases: Batch size too large can cause timeouts.

- **Split - id, title, desc**  
  - Type: SplitOut  
  - Config: Splits product object to extract `id`, `title`, and `description`.  
  - Inputs: Batch of products  
  - Outputs: To Number of Options node  
  - Edge cases: Missing fields cause null values.

- **Number of Options**  
  - Type: Set  
  - Config: Sets `number_of_options` to default "3" (string).  
  - Inputs: From Split - id, title, desc  
  - Outputs: To Calculate Options

- **Calculate Options**  
  - Type: Code  
  - Config: JavaScript code calculates `number_of_options = parseInt(input) + 1` and returns `result = number_of_options - 1`.  
  - Inputs: Number of Options  
  - Outputs: To If1 and later nodes  
  - Edge cases: Non-numeric input causes parse errors.

- **If1**  
  - Type: If  
  - Config: Checks if `result` equals 0; if yes, sends to Loop Over Items (reprocess), else to GS - Add Product Option.  
  - Inputs: Calculate Options output  
  - Outputs: Controls branching for processing or logging.

- **Remember Options**  
  - Type: Set  
  - Config: Stores `number_of_options` as numeric value for subsequent use.  
  - Inputs: Update Product Option node output  
  - Outputs: None direct; used for internal tracking.

- **Sticky Note1**  
  - Content: "Set the Number of Options you'd like for the Title and Description"

- **Sticky Note2**  
  - Content: "Process Title and Description Options"

---

#### 1.4 AI Content Generation

**Overview:**  
Uses OpenAI GPT-4o-mini model to generate optimized product titles and descriptions based on original content and brand instructions.

**Nodes Involved:**  
- Generate Title and Desc (OpenAI node)  
- Wikipedia (LangChain Wikipedia tool, AI helper)  
- Calculator (LangChain Calculator tool, AI helper)  
- Sticky Note (Sticky Note)

**Node Details:**  
- **Generate Title and Desc**  
  - Type: OpenAI (LangChain integration)  
  - Config:  
    - Model: GPT-4o-mini  
    - Prompt: Includes original product title and description, requests keyword extraction and new content.  
    - Brand Guidelines injected dynamically into prompt (brand name, tone, custom instructions).  
    - Output fields: `keyword`, `title`, `description` in JSON format.  
  - Inputs: From GS - Add Product Option (Google Sheet append)  
  - Outputs: To Update Product Option  
  - Edge cases: Rate limiting, API key issues, malformed response; must handle JSON parsing errors.

- **Wikipedia & Calculator**  
  - Type: LangChain tools  
  - Config: Attached as AI tools to assist Generate Title and Desc node if needed (e.g., keyword context or calculations).  
  - Inputs/Outputs: Internal to AI node; optional.

- **Sticky Note**  
  - Content: "Update Title and Description"

---

#### 1.5 Google Sheets Logging & Monitoring

**Overview:**  
Logs product data and AI-generated options to Google Sheets for tracking; monitors sheet for update triggers.

**Nodes Involved:**  
- GS - Add Product Option (Google Sheets)  
- Update Product Option (Google Sheets)  
- Google Sheets Trigger (Google Sheets Trigger)  
- Sticky Note9 (Sticky Note)

**Node Details:**  
- **GS - Add Product Option**  
  - Type: Google Sheets (Append)  
  - Config: Appends new rows with product info including original and processed titles/descriptions, status "Product Processing", random unique xid, date/time stamps.  
  - Inputs: From If1 node (products ready for processing)  
  - Outputs: To Generate Title and Desc  
  - Edge cases: Sheet permission issues, rate limits.

- **Update Product Option**  
  - Type: Google Sheets (Append or Update)  
  - Config: Updates rows by xid matching, sets status to "Option added", stores AI-generated keyword, titles, descriptions, and other product data.  
  - Inputs: From Generate Title and Desc output  
  - Outputs: To Remember Options  
  - Edge cases: Matching failures, concurrent edits.

- **Google Sheets Trigger**  
  - Type: Google Sheets Trigger  
  - Config: Watches for row updates in column "upload" every minute; sheet and doc IDs configured.  
  - Inputs: External (Google Sheets)  
  - Outputs: To If node for conditional update processing.  
  - Edge cases: Trigger delay, permissions.

- **Sticky Note9**  
  - Content: Personal branding and external links by Alex Kim, workflow author and architect.

---

#### 1.6 Conditional Logic & Update Control

**Overview:**  
Controls workflow branching based on product update readiness and upload flags.

**Nodes Involved:**  
- If (Google Sheets upload flag check)  
- If1 (Number of Options check)

**Node Details:**  
- **If (Google Sheets upload flag check)**  
  - Type: If  
  - Config: Checks if `upload` field equals "yes" in the updated sheet row.  
  - Inputs: Google Sheets Trigger  
  - Outputs: If yes, proceeds to Printify - Get Shops1 (update product), else ends.  
  - Edge cases: Missing or malformed upload flag.

- **If1**  
  - Already described in 1.3; controls continuation based on number_of_options calculation.

---

#### 1.7 Printify Product Update

**Overview:**  
Sends updated product titles and descriptions back to Printify via API.

**Nodes Involved:**  
- Printify - Get Shops1 (HTTP Request)  
- Printify - Update Product (HTTP Request)

**Node Details:**  
- **Printify - Get Shops1**  
  - Type: HTTP Request  
  - Config: GET shops list from Printify to confirm shop data before update.  
  - Inputs: From If (upload check)  
  - Outputs: To Printify - Update Product  
  - Edge cases: Auth failures, network issues.

- **Printify - Update Product**  
  - Type: HTTP Request (PUT)  
  - Config: Updates product at `https://api.printify.com/v1/shops/{{ shop_id }}/products/{{ product_id }}.json`  
  - Body includes updated `title` and `description` from Google Sheets data.  
  - Inputs: Shops list from Printify - Get Shops1 and product data from Google Sheets Trigger  
  - Outputs: None downstream  
  - Edge cases: Auth errors, rate limits, invalid product IDs, API response failures.

---

#### 1.8 Utility & Support Nodes

**Nodes Involved:**  
- Sticky Notes (various)  
- Calculator & Wikipedia (AI Tools)

**Details:**  
Sticky notes provide contextual information, instructions, and branding. Calculator and Wikipedia nodes augment AI processing but are optional in the current flow.

---

### 3. Summary Table

| Node Name                         | Node Type                 | Functional Role                            | Input Node(s)                         | Output Node(s)                       | Sticky Note                                                                                                   |
|----------------------------------|---------------------------|-------------------------------------------|-------------------------------------|------------------------------------|--------------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’     | Manual Trigger            | Manual start of workflow                   | -                                   | Brand Guidelines + Custom Instructions |                                                                                                              |
| Brand Guidelines + Custom Instructions | Set                       | Sets brand name, tone, custom instructions | Manual Trigger                      | Printify - Get Shops                | # Update your Brand Guidelines before running this workflow\nYou can also add custom instructions for the AI node. |
| Printify - Get Shops              | HTTP Request              | Retrieves Printify shops                   | Brand Guidelines + Custom Instructions | Printify - Get Products             | # You can swap out the API calls to similar services like Printful, Vistaprint, etc.                          |
| Printify - Get Products           | HTTP Request              | Retrieves products for each shop           | Printify - Get Shops                 | Split Out                         |                                                                                                              |
| Split Out                        | SplitOut                  | Splits product list into individual items | Printify - Get Products              | Loop Over Items                    |                                                                                                              |
| Loop Over Items                  | SplitInBatches            | Processes products in batches               | Split Out                          | (1) Split - id, title, desc (main)<br>(2) no output on second main | # Set the Number of Options you'd like for the Title and Description                                           |
| Split - id, title, desc          | SplitOut                  | Extracts id, title, description            | Loop Over Items                     | Number of Options                 |                                                                                                              |
| Number of Options                | Set                       | Sets default number of options for AI     | Split - id, title, desc             | Calculate Options                 |                                                                                                              |
| Calculate Options                | Code                      | Calculates and adjusts number of options  | Number of Options                   | If1                             |                                                                                                              |
| If1                             | If                        | Branches based on number_of_options result | Calculate Options                   | Loop Over Items (0 branch)<br>GS - Add Product Option (1 branch) |                                                                                                              |
| GS - Add Product Option          | Google Sheets             | Appends product info to Google Sheets     | If1                               | Generate Title and Desc           |                                                                                                              |
| Generate Title and Desc          | OpenAI (LangChain)        | Generates optimized title and description | GS - Add Product Option             | Update Product Option             | # Update Title and Description                                                                                |
| Update Product Option            | Google Sheets             | Updates Google Sheets with AI output      | Generate Title and Desc             | Remember Options                 |                                                                                                              |
| Remember Options                | Set                       | Stores final number_of_options value       | Update Product Option               | Calculate Options                |                                                                                                              |
| Google Sheets Trigger           | Google Sheets Trigger     | Watches Google Sheets for row updates      | External (Google Sheets)            | If                             | # AlexK1919 personal branding and resource links                                                             |
| If                             | If                        | Checks if product upload flag is "yes"    | Google Sheets Trigger               | Printify - Get Shops1             |                                                                                                              |
| Printify - Get Shops1            | HTTP Request              | Gets shops before product update           | If                               | Printify - Update Product         |                                                                                                              |
| Printify - Update Product        | HTTP Request (PUT)        | Updates product title and description      | Printify - Get Shops1               | None                            |                                                                                                              |
| Wikipedia                      | LangChain Wikipedia Tool  | Assists AI with knowledge                   | Attached AI tool                   | Attached AI tool                 |                                                                                                              |
| Calculator                    | LangChain Calculator Tool | Assists AI with calculations                 | Attached AI tool                   | Attached AI tool                 |                                                                                                              |
| Sticky Note1                   | Sticky Note               | Instruction on setting number of options   | -                                 | -                               | # Set the Number of Options you'd like for the Title and Description                                           |
| Sticky Note2                   | Sticky Note               | Instruction on processing title/desc       | -                                 | -                               | # Process Title and Description Options                                                                        |
| Sticky Note9                   | Sticky Note               | Author branding and workflow resources      | -                                 | -                               | # AlexK1919 bio, products used, and Google Sheets template link                                                |
| Sticky Note10                  | Sticky Note               | Alternative platform API swap suggestion    | -                                 | -                               | # You can swap out the API calls to similar services like Printful, Vistaprint, etc.                          |
| Sticky Note11                  | Sticky Note               | Reminder to update brand guidelines         | -                                 | -                               | # Update your Brand Guidelines before running this workflow\nYou can also add custom instructions for the AI node. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Name: "When clicking ‘Test workflow’"  
   - No parameters. Used to start the workflow manually.

2. **Create Set Node for Brand Guidelines**  
   - Type: Set  
   - Name: "Brand Guidelines + Custom Instructions"  
   - Assign variables:  
     - `brand_name`: "AlexK1919"  
     - `brand_tone`: "informal, instructional, trustoworthy"  
     - `custom_instructions`: "re-write for the coming Christmas season"

3. **Connect Manual Trigger → Brand Guidelines Node**

4. **Create HTTP Request Node to Fetch Shops**  
   - Type: HTTP Request  
   - Name: "Printify - Get Shops"  
   - Method: GET  
   - URL: `https://api.printify.com/v1/shops.json`  
   - Authentication: HTTP Header Auth with Printify API Key (credential setup required with correct API key in header).  
   - Connect Brand Guidelines → Printify - Get Shops

5. **Create HTTP Request Node to Fetch Products**  
   - Type: HTTP Request  
   - Name: "Printify - Get Products"  
   - Method: GET  
   - URL: Expression: `https://api.printify.com/v1/shops/{{ $json.id }}/products.json` (dynamic shop id)  
   - Authentication: Same Printify HTTP Header Auth credential  
   - Connect Printify - Get Shops → Printify - Get Products

6. **Create SplitOut Node to Separate Products**  
   - Type: Split Out  
   - Name: "Split Out"  
   - Field to split out: "data" (array of products)  
   - Connect Printify - Get Products → Split Out

7. **Create SplitInBatches Node for Batch Processing**  
   - Type: SplitInBatches  
   - Name: "Loop Over Items"  
   - Configure batch size as needed (default or user-defined)  
   - Connect Split Out → Loop Over Items

8. **Create SplitOut Node to Extract Product Fields**  
   - Type: Split Out  
   - Name: "Split - id, title, desc"  
   - Field to split out: "id"  
   - Include other fields: "title, description"  
   - Connect Loop Over Items → Split - id, title, desc

9. **Create Set Node for Number of Options**  
   - Type: Set  
   - Name: "Number of Options"  
   - Assign: `number_of_options` = "3" (string)  
   - Connect Split - id, title, desc → Number of Options

10. **Create Code Node to Calculate Options**  
    - Type: Code  
    - Name: "Calculate Options"  
    - JavaScript:  
      ```js
      const inputData = $json["number_of_options"];
      const initialValue = parseInt(inputData, 10);
      const numberOfOptions = initialValue + 1;
      const result = numberOfOptions - 1;
      return {
        number_of_options: numberOfOptions,
        result,
      };
      ```
    - Connect Number of Options → Calculate Options

11. **Create If Node to Check Result**  
    - Type: If  
    - Name: "If1"  
    - Condition: `result` equals 0 (number)  
    - True output branch: Connect back to Loop Over Items (to reprocess)  
    - False output branch: Connect to "GS - Add Product Option" node (next step)  
    - Connect Calculate Options → If1

12. **Create Google Sheets Append Node to Log Initial Product Data**  
    - Type: Google Sheets  
    - Name: "GS - Add Product Option"  
    - Operation: Append  
    - Document ID and Sheet name: Set to your Google Sheets document and sheet  
    - Columns to include: xid (random string), date, time, status ("Product Processing"), product_id, original_title, product_title, original_desc, product_desc, product_url, image_url, video_url  
    - Connect If1 (False branch) → GS - Add Product Option

13. **Create OpenAI Node to Generate Title and Description**  
    - Type: OpenAI (LangChain)  
    - Name: "Generate Title and Desc"  
    - Model: GPT-4o-mini  
    - Messages:  
      - System and assistant messages including brand guidelines, custom instructions, original title/description, and prompt to generate keyword, new title, and description.  
    - Credentials: Connect OpenAI API Key credentials  
    - Connect GS - Add Product Option → Generate Title and Desc

14. **Create Google Sheets AppendOrUpdate Node to Save AI Output**  
    - Type: Google Sheets  
    - Name: "Update Product Option"  
    - Operation: Append or Update (match by xid)  
    - Columns: Include xid, status ("Option added"), keyword, product_id, product_title, product_desc, original_title, original_desc, product_url, image_url, video_url  
    - Connect Generate Title and Desc → Update Product Option

15. **Create Set Node to Remember Number of Options**  
    - Type: Set  
    - Name: "Remember Options"  
    - Assign `number_of_options` as numeric from Calculate Options result minus 1  
    - Connect Update Product Option → Remember Options

16. **Create Google Sheets Trigger Node to Monitor Uploads**  
    - Type: Google Sheets Trigger  
    - Name: "Google Sheets Trigger"  
    - Configure to watch "upload" column changes every minute on your Google Sheets doc and sheet  
    - Credentials: Google Sheets OAuth2 credentials setup required

17. **Create If Node to Check Upload Flag**  
    - Type: If  
    - Name: "If"  
    - Condition: Check if `upload` field equals "yes"  
    - Connect Google Sheets Trigger → If

18. **Create HTTP Request Node to Get Shops (Update Phase)**  
    - Type: HTTP Request  
    - Name: "Printify - Get Shops1"  
    - GET `https://api.printify.com/v1/shops.json` with Printify HTTP Header Auth  
    - Connect If (True branch) → Printify - Get Shops1

19. **Create HTTP Request Node to Update Product**  
    - Type: HTTP Request (PUT)  
    - Name: "Printify - Update Product"  
    - URL: `https://api.printify.com/v1/shops/{{ $json.id }}/products/{{ $('Google Sheets Trigger').item.json.product_id }}.json`  
    - Body: JSON with updated `title` and `description` from Google Sheets Trigger data  
    - Authentication: Printify HTTP Header Auth  
    - Connect Printify - Get Shops1 → Printify - Update Product

20. **Add Sticky Notes** for instructions and branding as per the original workflow for user clarity.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                       | Context or Link                                                                                                                           |
|----------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------|
| Author: Alex Kim — AI-Native Workflow Automation Architect. Profile and bio: https://beacons.ai/alexk1919                                        | Sticky Note9                                                                                                                             |
| Google Sheets Template for this workflow: https://docs.google.com/spreadsheets/d/12Y7M5YSUW1e8UUOjupzctOrEtgMK-0Wb32zcVpNcfjk/edit#gid=0          | Sticky Note9                                                                                                                             |
| Workflow supports swapping Printify API calls to similar services such as Printful or Vistaprint                                                  | Sticky Note10                                                                                                                            |
| Update brand guidelines and custom instructions before running the workflow to ensure tone consistency                                            | Sticky Note11                                                                                                                            |
| Use batch processing (Loop Over Items node) to optimize performance for large product catalogs                                                    | General best practice                                                                                                                    |
| Configure retries and error logging on HTTP Request nodes to handle API rate limits and network errors                                           | Recommended for production stability                                                                                                    |
| OpenAI API usage requires valid API key with sufficient quota; monitor usage to avoid unexpected costs                                           | Credential setup advice                                                                                                                  |
| Google Sheets integration requires the sheet to be shared with the service account and proper OAuth2 credentials configured in n8n              | Setup instructions                                                                                                                      |
| Use Google Sheets as a single source of truth for tracking product update status and content revisions                                           | Workflow design principle                                                                                                               |

---

This reference document fully describes the “Optimize & Update Printify Title and Description Workflow,” enabling understanding, reproduction, and modification without requiring the original JSON export.