Generate Personalized Sales Emails from Google Maps with OpenAI to Google Sheets

https://n8nworkflows.xyz/workflows/generate-personalized-sales-emails-from-google-maps-with-openai-to-google-sheets-11028


# Generate Personalized Sales Emails from Google Maps with OpenAI to Google Sheets

### 1. Workflow Overview

This workflow automates personalized sales email generation targeting local businesses found via Google Maps. It is designed for sales professionals, marketing agencies, and freelancers who want to streamline lead generation and tailored outreach.

**Logical Blocks:**

- **1.1 Configuration Setup:** Defines search parameters, scraping limits, and product/service details.
- **1.2 Lead Scraping:** Invokes Apify’s Google Maps Scraper actor to retrieve business listings based on the query.
- **1.3 Iteration & Filtering:** Processes each lead individually, filtering those with websites.
- **1.4 Website Content Extraction:** Fetches website HTML and extracts relevant text snippets for context.
- **1.5 AI-Powered Email Generation:** Uses OpenAI to create personalized sales emails based on extracted data and service info.
- **1.6 Data Persistence:** Appends all collected and generated data into a structured Google Sheet for record-keeping and follow-up.

---

### 2. Block-by-Block Analysis

#### 2.1 Configuration Setup

- **Overview:** Initializes workflow variables such as search criteria, maximum number of places to scrape, Apify actor ID, and service details used in email generation.
- **Nodes Involved:**  
  - Manual Trigger  
  - Workflow Configuration  
- **Node Details:**

  - **Manual Trigger**  
    - Type: Trigger node to start workflow manually.  
    - Configuration: Default (no parameters).  
    - Inputs: None  
    - Outputs: Triggers "Workflow Configuration" node.  
    - Failures: None expected unless manual start is omitted.

  - **Workflow Configuration**  
    - Type: Set node, assigning static parameters.  
    - Key parameters:  
      - `searchQuery`: "渋谷 カフェ" (Shibuya Cafes) as example search keyword.  
      - `maxPlaces`: 5 (limits number of scraped businesses).  
      - `apifyActorId`: "compass/google-maps-scraper" (Apify actor ID).  
      - `serviceName`: "飲食店向け予約管理システム「yoyaku-kun」" (Reservation system product name).  
      - `serviceStrength`: Describes product USP detailing features like no phone reservation handling and organic ingredient page creation.  
    - Inputs: From Manual Trigger.  
    - Outputs: To Apify actor invocation node ("Get user runs list").  
    - Edge cases: Ensure inputs are valid strings/numbers; missing or malformed values cause downstream errors.

#### 2.2 Lead Scraping

- **Overview:** Calls Apify actor to scrape Google Maps for businesses matching the query, fetching a list of leads.
- **Nodes Involved:**  
  - Get user runs list (Apify actor node)  
  - Loop Over Items (splitInBatches node)  
- **Node Details:**

  - **Get user runs list**  
    - Type: Apify node invoking the "Actor runs" resource.  
    - Configuration: Uses credentials for Apify account; triggers scraping based on parameters from configuration node.  
    - Inputs: From "Workflow Configuration".  
    - Outputs: List of scraped businesses, passed to "Loop Over Items".  
    - Edge cases: API call failures, invalid actor ID, rate limits.

  - **Loop Over Items**  
    - Type: splitInBatches node to process each business one by one, enabling sequential downstream processing.  
    - Inputs: List of businesses from Apify.  
    - Outputs: Each item iteratively passed to filter and processing nodes.  
    - Edge cases: Empty input array, batch processing errors.

#### 2.3 Iteration & Filtering

- **Overview:** Filters out businesses without website URLs to focus on those with web presence.
- **Nodes Involved:**  
  - Check Website URL Exists (If node)  
- **Node Details:**

  - **Check Website URL Exists**  
    - Type: If node with condition checking non-empty `website` field.  
    - Configuration: Condition tests if website URL string is not empty.  
    - Inputs: Individual business item from Loop Over Items.  
    - Outputs: True branch to "Fetch Website HTML" if website exists; False branch ignored (no further processing).  
    - Edge cases: Missing or malformed URLs, false negatives if URL field is present but invalid.

#### 2.4 Website Content Extraction

- **Overview:** Fetches the HTML content of the business website and extracts meaningful text snippets to provide context for AI.
- **Nodes Involved:**  
  - Fetch Website HTML (HTTP Request node)  
  - Extract Text from HTML (Code node)  
- **Node Details:**

  - **Fetch Website HTML**  
    - Type: HTTP Request node  
    - Configuration: Performs GET request to the business website URL dynamically taken from input data.  
    - Inputs: From "Check Website URL Exists" (True branch).  
    - Outputs: Raw HTML content of the website.  
    - Edge cases: HTTP errors (404, 500), timeouts, redirects, SSL issues.

  - **Extract Text from HTML**  
    - Type: Code node (JavaScript)  
    - Configuration: Runs per item; strips scripts and styles, removes HTML tags, normalizes whitespace and trims text to 2000 characters. Also copies relevant fields from original item (title, category, website, address, phone).  
    - Inputs: Raw HTML from "Fetch Website HTML".  
    - Outputs: Cleaned site text and metadata for AI processing.  
    - Edge cases: Unexpected HTML structure, empty pages, code errors.

#### 2.5 AI-Powered Email Generation

- **Overview:** Uses OpenAI via LangChain node to generate personalized sales email content based on extracted website text and configured service details.
- **Nodes Involved:**  
  - Generate Personalized Email (OpenAI node)  
- **Node Details:**

  - **Generate Personalized Email**  
    - Type: OpenAI (LangChain) node configured for chat/message operation.  
    - Configuration: Uses OpenAI API credentials; prompt context includes website text and service description to craft an email.  
    - Inputs: Extracted site text and business metadata from previous node.  
    - Outputs: AI-generated email subject and body content.  
    - Edge cases: API failures, rate limits, prompt errors, unexpected or irrelevant output.

#### 2.6 Data Persistence

- **Overview:** Appends or updates a Google Sheet with all relevant business and email information for record-keeping.
- **Nodes Involved:**  
  - Save to Google Sheets (Google Sheets node)  
- **Node Details:**

  - **Save to Google Sheets**  
    - Type: Google Sheets node  
    - Configuration:  
      - Operation: appendOrUpdate  
      - Sheet name and Document ID must be set by user  
      - Maps fields: Store name, address, website, phone, extracted site info, generated email subject and body.  
      - Matching column for update: Store name  
    - Inputs: AI-generated email and business data.  
    - Outputs: None (final step).  
    - Edge cases: Credential failures, sheet access denied, incorrect sheet/document IDs, data mapping errors.

---

### 3. Summary Table

| Node Name                | Node Type                     | Functional Role                         | Input Node(s)           | Output Node(s)            | Sticky Note                                                                                                                             |
|--------------------------|-------------------------------|---------------------------------------|-------------------------|---------------------------|-----------------------------------------------------------------------------------------------------------------------------------------|
| Manual Trigger           | manualTrigger                  | Start workflow                        | None                    | Workflow Configuration    |                                                                                                                                         |
| Workflow Configuration   | set                           | Define search & service parameters    | Manual Trigger          | Get user runs list        | 1. **Configuration**: You define your search query (e.g., "Gyms in London"), the number of leads to fetch, and details about your service. |
| Get user runs list       | apify                         | Scrape Google Maps business leads     | Workflow Configuration  | Loop Over Items           | 2. **Lead Scraping**: The workflow triggers an Apify actor (Google Maps Scraper) to find businesses matching your criteria.             |
| Loop Over Items          | splitInBatches                | Iterate over each scraped lead         | Get user runs list       | Check Website URL Exists, Save to Google Sheets |                                                                                                                                         |
| Check Website URL Exists | if                            | Filter leads having website URLs      | Loop Over Items         | Fetch Website HTML        | 3. **Website Analysis**: It checks if the business has a website, fetches the HTML, and extracts relevant text to understand the business's vibe and offerings. |
| Fetch Website HTML       | httpRequest                   | Download business website HTML        | Check Website URL Exists | Extract Text from HTML    |                                                                                                                                         |
| Extract Text from HTML   | code                          | Clean and extract text content from HTML | Fetch Website HTML    | Generate Personalized Email |                                                                                                                                         |
| Generate Personalized Email | openAi (LangChain)          | Generate personalized sales email     | Extract Text from HTML  | Save to Google Sheets     | 4. **AI Email Generation**: OpenAI analyzes the scraped website text and generates a specific, personalized email subject and body promoting your service. |
| Save to Google Sheets    | googleSheets                  | Save business and email data to sheet | Generate Personalized Email, Loop Over Items | None                      | 5. **Data Storage**: All data (Business Name, Phone, Address, Website, Scraped Info, and Email Draft) is appended to a Google Sheet.     |
| Sticky Note              | stickyNote                    | Documentation and explanation notes   | None                    | None                      | See detailed notes in Section 5.                                                                                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger node:**  
   - Type: Manual Trigger  
   - Purpose: Manual start point for the workflow.

2. **Add Set node named "Workflow Configuration":**  
   - Assign variables:  
     - `searchQuery`: e.g., "渋谷 カフェ"  
     - `maxPlaces`: 5  
     - `apifyActorId`: "compass/google-maps-scraper"  
     - `serviceName`: e.g., "飲食店向け予約管理システム「yoyaku-kun」"  
     - `serviceStrength`: Your unique selling proposition string  
   - Connect Manual Trigger → Workflow Configuration.

3. **Add Apify node "Get user runs list":**  
   - Resource: Actor runs  
   - Credentials: Apify account credentials (set up in n8n)  
   - Configure to run the Google Maps Scraper actor with parameters from the set node.  
   - Connect Workflow Configuration → Get user runs list.

4. **Add splitInBatches node "Loop Over Items":**  
   - Purpose: Iterate over each scraped lead one-by-one.  
   - Connect Get user runs list → Loop Over Items.

5. **Add If node "Check Website URL Exists":**  
   - Condition: Check if `website` field is not empty (`{{$json.website}}` not empty).  
   - Connect Loop Over Items → Check Website URL Exists.

6. **Add HTTP Request node "Fetch Website HTML":**  
   - Method: GET  
   - URL: Set dynamically from `{{$json.website}}`.  
   - Connect If node True branch → Fetch Website HTML.

7. **Add Code node "Extract Text from HTML":**  
   - JavaScript code to:  
     - Remove `<script>` and `<style>` tags from HTML  
     - Strip all HTML tags  
     - Normalize whitespace  
     - Truncate to 2000 characters  
     - Copy relevant fields (title, categoryName, website, address, phone) from current item  
   - Connect Fetch Website HTML → Extract Text from HTML.

8. **Add OpenAI (LangChain) node "Generate Personalized Email":**  
   - Operation: message/chat  
   - Credentials: OpenAI API key configured in n8n.  
   - Prompt: Include extracted site text, business name, and your service info to generate personalized email subject and body.  
   - Connect Extract Text from HTML → Generate Personalized Email.

9. **Add Google Sheets node "Save to Google Sheets":**  
   - Operation: appendOrUpdate  
   - Configure:  
     - Document ID: Your Google Sheets file ID  
     - Sheet Name: Target sheet name (e.g., "Sheet1")  
     - Mapping columns:  
       - 店舗名 (Store Name) ← `{{$json.title}}`  
       - 住所 (Address) ← `{{$json.address}}`  
       - Webサイト (Website) ← `{{$json.website}}`  
       - 電話番号 (Phone) ← `{{$json.phone}}`  
       - サイトから取得した情報 (Info from Website) ← `{{$json.site_text}}`  
       - 生成されたメール件名 (Generated Email Subject) ← `{{$json.message.content}}`  
       - 生成されたメール本文 (Generated Email Body) ← `{{$json.message.content}}`  
     - Matching column for updates: 店舗名 (Store Name)  
   - Connect Generate Personalized Email → Save to Google Sheets.  
   - Also connect Loop Over Items → Save to Google Sheets (for items filtered out? Confirm logic).

10. **Verify all connections and node parameters.**  
11. **Set up credentials in n8n:**  
    - Apify API key  
    - OpenAI API key  
    - Google Sheets OAuth2 credentials with access to target spreadsheet.  
12. **Create the Google Sheet with required headers as above in the first row.**  
13. **Test run the workflow manually.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | Context or Link                                                                                     |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| This workflow automates lead generation and personalized email drafting by integrating Apify Google Maps Scraper, website content extraction, OpenAI email generation, and Google Sheets storage. It drastically reduces manual research and outreach efforts.                                                                                                                                                                                                                                                         | Overall workflow purpose and benefits.                                                            |
| Setup requires accounts and API credentials for Apify, OpenAI, and Google Sheets API enabled in Google Cloud Platform.                                                                                                                                                                                                                                                                                                                                                                                             | Credential setup prerequisites.                                                                   |
| Google Sheet must have columns with Japanese headers matching the mapping: 店舗名, 住所, Webサイト, 電話番号, サイトから取得した情報, 生成されたメール件名, 生成されたメール本文.                                                                                                                                                                                                                                                                                                                                  | Google Sheets template requirements.                                                             |
| The OpenAI prompt can be customized in the "Generate Personalized Email" node to change tone, language, and structure of the sales email.                                                                                                                                                                                                                                                                                                                                                                          | Customization tip for email style and content.                                                   |
| Filtering leads without websites helps focus on businesses with an online presence, improving email relevance. This is done in the "Check Website URL Exists" node.                                                                                                                                                                                                                                                                                                                                                  | Filtering logic explanation.                                                                      |
| Adjust `maxPlaces` in configuration to control number of leads per run, managing API usage and workflow duration.                                                                                                                                                                                                                                                                                                                                                                                                     | Usage and cost control tip.                                                                        |
| Sticky notes in the workflow provide detailed explanations and setup instructions, including step-by-step guidance and contextual information.                                                                                                                                                                                                                                                                                                                                                                       | In-workflow documentation.                                                                        |
| For troubleshooting: verify all API keys, check rate limits, and ensure Google Sheets have correct permissions. Validate website URLs to avoid HTTP errors in fetching HTML.                                                                                                                                                                                                                                                                                                                                         | Common error sources and debugging tips.                                                         |
| Apify Google Maps Scraper actor reference: https://apify.com/compass/google-maps-scraper                                                                                                                                                                                                                                                                                                                                                                                                                              | External resource link for actor details.                                                        |
| OpenAI API documentation: https://platform.openai.com/docs/api-reference/introduction                                                                                                                                                                                                                                                                                                                                                                                                                                  | Reference for AI node configuration.                                                             |
| Google Sheets API documentation: https://developers.google.com/sheets/api                                                                                                                                                                                                                                                                                                                                                                                                                                               | Reference for Google Sheets integration.                                                         |

---

**Disclaimer:** The provided text is derived exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly available.