Automatically Search Facebook Ad Products on Amazon using Apify Scrapers

https://n8nworkflows.xyz/workflows/automatically-search-facebook-ad-products-on-amazon-using-apify-scrapers-6888


# Automatically Search Facebook Ad Products on Amazon using Apify Scrapers

### 1. Workflow Overview

This n8n workflow automates the process of extracting product information from Facebook Ads and searching for corresponding products on Amazon using Apify scrapers. It is designed primarily for use cases like Amazon FBA sellers, dropshippers, or e-commerce analysts who want to quickly map Facebook ad products to Amazon listings.

The workflow is logically divided into the following blocks, each representing a key functional stage:

- **1.1 Facebook Ads Extraction:** Trigger the Facebook Ad Library scraper to fetch active ads based on a specified keyword and retrieve the results.
- **1.2 Facebook Ad Data Preparation:** Save and prepare Facebook ad metadata and URLs for further processing.
- **1.3 Web Content Scraping:** Scrape website content from URLs extracted from Facebook ads.
- **1.4 Product Name Identification:** Use OpenAI GPT-4.1 to analyze scraped website content or fallback Facebook ad data to identify the product name to search on Amazon.
- **1.5 Amazon Product Search:** Trigger the Amazon Search scraper with the identified product name and retrieve search results.
- **1.6 Post-Processing and Filtering:** Clean and structure Amazon search results, filter by product rating, limit results, and prepare data for storage.
- **1.7 Data Storage:** Optionally append the filtered product data into a Google Sheet for record-keeping.

---

### 2. Block-by-Block Analysis

#### 2.1 Facebook Ads Extraction

- **Overview:**  
  Initiates the scraping of Facebook ads by calling the Apify Facebook Ad Library scraper API with a specific URL and parameters, then retrieves the scraped data.

- **Nodes Involved:**  
  - Manual Trigger  
  - Run FB Library Actor (HTTP Request - POST)  
  - Get Actor's Scraped Content (HTTP Request - GET)  
  - SaveFBData (Set)  

- **Node Details:**

  - **Manual Trigger**  
    - Type: Manual trigger node to start the workflow manually.  
    - Configuration: Default, no parameters.  
    - Connections: Outputs to Run FB Library Actor.  
    - Edge Cases: None; manual start only.

  - **Run FB Library Actor**  
    - Type: HTTP Request (POST)  
    - Role: Calls the Apify Facebook Ad Library scraper's "Run Actor synchronously" API endpoint.  
    - Configuration:  
      - POST method with JSON body specifying:  
        - count: 10 ads  
        - scrapeAdDetails: true  
        - activeStatus: all  
        - URLs array with the Facebook Ad Library URL for searching "Wireless Mouse" ads in the US.  
    - Connections: Input from Manual Trigger; outputs to Get Actor's Scraped Content.  
    - Edge Cases: API authentication errors, malformed URL, request timeouts, or empty results.

  - **Get Actor's Scraped Content**  
    - Type: HTTP Request (GET)  
    - Role: Fetches the latest dataset items from the Apify actor run.  
    - Configuration: Default GET request to Apify dataset endpoint.  
    - Connections: Input from Run FB Library Actor; outputs to SaveFBData.  
    - Edge Cases: API call failures, empty dataset, pagination issues.

  - **SaveFBData**  
    - Type: Set node  
    - Role: Extracts and saves relevant Facebook ad snapshot data fields for downstream use.  
    - Configuration: Sets fields like `snapshot.page_name`, `snapshot.page_profile_uri`, `snapshot.caption` (URL), and `snapshot.body.text` from the fetched data.  
    - Connections: Input from Get Actor's Scraped Content; outputs to Loop Over Items.  
    - Edge Cases: Missing or incomplete snapshot data.

---

#### 2.2 Facebook Ad Data Loop and Conditional Processing

- **Overview:**  
  Loops over each Facebook ad item, checking if an Amazon URL was found previously. If yes, save to Google Sheets; if no, scrape the website content to find the product.

- **Nodes Involved:**  
  - Loop Over Items (SplitInBatches)  
  - If Amazon URL Found (IF node)  
  - Append row in sheet (Google Sheets)  
  - Scrape Website Content (HTTP Request GET)  

- **Node Details:**

  - **Loop Over Items**  
    - Type: SplitInBatches  
    - Role: Processes Facebook ad items one by one to handle large datasets efficiently.  
    - Configuration: Default batch size; error handling set to continue.  
    - Connections: Input from SaveFBData and SaveItems; outputs to If Amazon URL Found and Scrape Website Content.  
    - Edge Cases: Batch size limits, partial failures.

  - **If Amazon URL Found**  
    - Type: IF node  
    - Role: Checks if the current item contains a non-empty Amazon URL field.  
    - Configuration: Condition tests if `amazonURL` is not empty.  
    - Connections: True branch to Append row in sheet; False branch to Scrape Website Content.  
    - Edge Cases: Empty or malformed URLs.

  - **Append row in sheet**  
    - Type: Google Sheets Append  
    - Role: Appends the product data row into a specified Google Sheet.  
    - Configuration:  
      - Document ID and sheet tab specified.  
      - Columns mapped with product price, rating, Amazon URL, Facebook page name, etc.  
      - Credentials: Google Sheets OAuth2.  
    - Connections: Input from If Amazon URL Found true branch.  
    - Edge Cases: Authentication failures, quota limits, sheet structure changes.

  - **Scrape Website Content**  
    - Type: HTTP Request (GET)  
    - Role: Retrieves the HTML content from the Facebook ad’s website URL (`snapshot.caption` field).  
    - Configuration: URL dynamic expression `={{ $json.snapshot.caption }}`; error handling set to continue.  
    - Connections: Input from If Amazon URL Found false branch; outputs to Turn Into PlainText.  
    - Edge Cases: Broken links, timeouts, 404 errors.

---

#### 2.3 HTML Content Processing and Product Name Identification

- **Overview:**  
  Converts scraped HTML content into clean plain text, then uses OpenAI GPT to identify the product name for Amazon search.

- **Nodes Involved:**  
  - Turn Into PlainText (Code node)  
  - Product Name Finder (OpenAI GPT node)  
  - If No Product Found (IF node)  

- **Node Details:**

  - **Turn Into PlainText**  
    - Type: Code (JavaScript)  
    - Role: Cleans HTML to plain text by stripping tags, decoding entities, and formatting text.  
    - Configuration: Custom JS code that handles removing scripts/styles, replacing list items, decoding HTML entities, and trimming whitespace.  
    - Connections: Input from Scrape Website Content; outputs to Product Name Finder.  
    - Edge Cases: Empty or malformed HTML, script errors, encoding issues.

  - **Product Name Finder**  
    - Type: OpenAI GPT (n8n-nodes-base.langchain.openAi)  
    - Role: Uses GPT-4.1-mini model to extract the most likely product name from the scraped content or fallback Facebook ad data.  
    - Configuration:  
      - System prompt instructs to find a single product name suitable for Amazon search.  
      - Inputs: scraped plainText, FB ad page name, caption, and body text via expressions.  
      - Credentials: OpenAI API key.  
    - Connections: Input from Turn Into PlainText; outputs to If No Product Found.  
    - Edge Cases: API rate limits, empty input content, ambiguous product descriptions.

  - **If No Product Found**  
    - Type: IF node  
    - Role: Checks if the GPT output equals "No Product Found".  
    - Configuration: String equality check on GPT response content.  
    - Connections: True branch loops back to Loop Over Items to retry or proceed; False branch triggers Run Amazon Search Actor.  
    - Edge Cases: GPT generating unexpected output or empty string.

---

#### 2.4 Amazon Product Search and Result Processing

- **Overview:**  
  Searches Amazon for the identified product name using Apify’s Amazon Search scraper, processes results, filters by rating, limits results, and prepares data for storage.

- **Nodes Involved:**  
  - Run Amazon Search Actor (HTTP Request POST)  
  - Get Amazon Search Results (HTTP Request GET)  
  - Edit Fields1 (Set)  
  - Filter (Filter node)  
  - Limit (Limit node)  
  - SaveItems (Set)  

- **Node Details:**

  - **Run Amazon Search Actor**  
    - Type: HTTP Request (POST)  
    - Role: Calls Apify Amazon Search scraper with the product name from GPT output.  
    - Configuration:  
      - JSON body uses the product name as keyword, domainCode "com", sorted by recent, maxPages 1, category "aps".  
    - Connections: Input from If No Product Found false branch; outputs to Get Amazon Search Results.  
    - Edge Cases: API failures, invalid product names.

  - **Get Amazon Search Results**  
    - Type: HTTP Request (GET)  
    - Role: Retrieves dataset results from the Amazon Search scraper run.  
    - Configuration: Default GET request to Apify dataset endpoint.  
    - Connections: Input from Run Amazon Search Actor; outputs to Edit Fields1.  
    - Edge Cases: Empty results, API errors.

  - **Edit Fields1**  
    - Type: Set node  
    - Role: Transforms and cleans Amazon search result fields for later use.  
    - Configuration:  
      - Creates `amazonURL` by concatenating "https://www.amazon.com" with the product’s dpUrl.  
      - Extracts productTitle, price, and trims productRating to first 3 characters.  
    - Connections: Input from Get Amazon Search Results; outputs to Filter.  
    - Edge Cases: Missing or malformed fields.

  - **Filter**  
    - Type: Filter node  
    - Role: Filters products to those with ratings greater than or equal to 4.3.  
    - Configuration: Numeric condition on `productRating >= 4.3`.  
    - Connections: Input from Edit Fields1; outputs to Limit.  
    - Edge Cases: Non-numeric or missing ratings.

  - **Limit**  
    - Type: Limit node  
    - Role: Limits the number of Amazon search results to 1 to avoid duplicates.  
    - Configuration: Default limit 1.  
    - Connections: Input from Filter; outputs to SaveItems.  
    - Edge Cases: No results after filtering.

  - **SaveItems**  
    - Type: Set node  
    - Role: Prepares final product data fields for looping back to Loop Over Items.  
    - Configuration: Sets fields amazonURL, productTitle, price, productRating.  
    - Connections: Input from Limit; outputs back to Loop Over Items.  
    - Edge Cases: Data integrity issues.

---

### 3. Summary Table

| Node Name               | Node Type                    | Functional Role                                  | Input Node(s)               | Output Node(s)              | Sticky Note                                                  |
|-------------------------|------------------------------|-------------------------------------------------|-----------------------------|-----------------------------|--------------------------------------------------------------|
| Manual Trigger          | Manual Trigger               | Starts workflow manually                         |                             | Run FB Library Actor         | # 👋 Introduction describes workflow and Apify scrapers setup |
| Run FB Library Actor    | HTTP Request (POST)          | Calls Facebook Ad Library scraper API           | Manual Trigger              | Get Actor's Scraped Content  | 📤 Extract FB Ad Library explains API call details            |
| Get Actor's Scraped Content | HTTP Request (GET)         | Retrieves scraped Facebook ads data              | Run FB Library Actor        | SaveFBData                  | 📤 Extract FB Ad Library explains API call details            |
| SaveFBData              | Set                         | Saves Facebook ad snapshot data                  | Get Actor's Scraped Content | Loop Over Items             |                                                              |
| Loop Over Items         | SplitInBatches              | Loops over Facebook ad items                      | SaveFBData, SaveItems       | If Amazon URL Found, Scrape Website Content |                                                              |
| If Amazon URL Found     | IF                          | Checks if Amazon URL exists                        | Loop Over Items             | Append row in sheet, Scrape Website Content |                                                              |
| Append row in sheet     | Google Sheets Append        | Saves product data into Google Sheets             | If Amazon URL Found (true)  | Loop Over Items             | 📥 Insert Products Into Sheet notes optional sheet usage      |
| Scrape Website Content  | HTTP Request (GET)           | Scrapes website content from Facebook ad URL     | If Amazon URL Found (false) | Turn Into PlainText         | 🔎 Find Product Name explains scraping from FB ad URL        |
| Turn Into PlainText     | Code                        | Converts HTML content to clean plain text         | Scrape Website Content      | Product Name Finder         | 🔎 Find Product Name explains scraping from FB ad URL        |
| Product Name Finder     | OpenAI GPT                  | Extracts product name from website or FB data     | Turn Into PlainText         | If No Product Found         |                                                              |
| If No Product Found     | IF                          | Checks if product name extraction failed          | Product Name Finder         | Loop Over Items, Run Amazon Search Actor |                                                              |
| Run Amazon Search Actor | HTTP Request (POST)          | Calls Amazon Search scraper API                    | If No Product Found (false) | Get Amazon Search Results   | 🔎 Find Product on Amazon explains API call details           |
| Get Amazon Search Results | HTTP Request (GET)          | Retrieves Amazon search results                     | Run Amazon Search Actor     | Edit Fields1                | 🔎 Find Product on Amazon explains API call details           |
| Edit Fields1            | Set                         | Cleans and formats Amazon product fields           | Get Amazon Search Results   | Filter                     |                                                              |
| Filter                  | Filter                      | Filters products by rating >= 4.3                   | Edit Fields1                | Limit                      | Limit node sticky note explains limiting search results to 1  |
| Limit                   | Limit                       | Limits results to 1 to avoid duplicates             | Filter                      | SaveItems                  | Limit node sticky note explains limiting search results to 1  |
| SaveItems               | Set                         | Prepares final product data for next processing     | Limit                      | Loop Over Items             |                                                              |
| Sticky Note             | Sticky Note                 | Provides documentation and guidance                 |                             |                             | Multiple sticky notes provide detailed explanations and images |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node** to start the workflow manually.

2. **Add an HTTP Request node named "Run FB Library Actor":**  
   - Method: POST  
   - URL: Apify "Run Actor synchronously" endpoint for Facebook Ad Library scraper  
   - Body (JSON):  
     ```json
     {
       "count": 10,
       "scrapeAdDetails": true,
       "scrapePageAds.activeStatus": "all",
       "urls": [
         {
           "url": "https://www.facebook.com/ads/library/?active_status=active&ad_type=all&country=US&is_targeted_country=false&media_type=all&q=Wireless%20Mouse&search_type=keyword_unordered&source=fb-logo",
           "method": "GET"
         }
       ]
     }
     ```
   - Send Body: JSON

3. **Connect "Run FB Library Actor" to an HTTP Request node "Get Actor's Scraped Content":**  
   - Method: GET  
   - URL: Apify dataset endpoint to fetch last run results of the Facebook scraper.

4. **Add a Set node "SaveFBData" to extract and store Facebook ad snapshot fields:**  
   - Assign fields from incoming JSON, e.g.:  
     - `snapshot.page_name`  
     - `snapshot.page_profile_uri`  
     - `snapshot.caption` (URL of ad link)  
     - `snapshot.body.text`  

5. **Add a SplitInBatches node "Loop Over Items" to process each Facebook ad one-by-one.**

6. **Add an IF node "If Amazon URL Found":**  
   - Condition: `amazonURL` field is not empty.  
   - True branch connects to "Append row in sheet".  
   - False branch connects to "Scrape Website Content".

7. **Create a Google Sheets node "Append row in sheet":**  
   - Operation: Append  
   - Document: Select your Google Sheets document (authenticate with Google OAuth2).  
   - Sheet: Select appropriate tab.  
   - Map columns: Price, Rating, Link URL, AmazonURL, FB Page Name, ProductTitle from current JSON or from "SaveFBData" node.

8. **Add an HTTP Request node "Scrape Website Content":**  
   - Method: GET  
   - URL: Expression `={{ $json.snapshot.caption }}` (the URL from the Facebook ad).  
   - Set error mode to "continue".

9. **Add a Code node "Turn Into PlainText":**  
   - Paste the provided JavaScript code that converts HTML to clean plain text.  
   - Input: HTML content from "Scrape Website Content" node.

10. **Add an OpenAI GPT node "Product Name Finder":**  
    - Model: GPT-4.1-mini (or equivalent)  
    - System prompt: Supply the content of the scraped website and fallback FB ad data, instructing to identify a single product name suitable for Amazon search, else output "No Product Found".  
    - Inputs:  
      - Website plainText from "Turn Into PlainText"  
      - FB Page Name, Caption, Body Text from "SaveFBData"  
    - Authenticate with OpenAI API key.

11. **Add an IF node "If No Product Found":**  
    - Condition: Check if GPT output equals "No Product Found".  
    - True branch loops back to "Loop Over Items" to continue with next item.  
    - False branch connects to "Run Amazon Search Actor".

12. **Add HTTP Request node "Run Amazon Search Actor":**  
    - Method: POST  
    - URL: Apify "Run Actor synchronously" endpoint for Amazon Search scraper.  
    - Body (JSON): Use expression to insert product name from OpenAI output:  
      ```json
      {
        "input": [
          {
            "keyword": "{{ $json.message.content }}",
            "domainCode": "com",
            "sortBy": "recent",
            "maxPages": 1,
            "category": "aps"
          }
        ]
      }
      ```
    - Send Body: JSON

13. **Add HTTP Request node "Get Amazon Search Results":**  
    - Method: GET  
    - URL: Apify dataset endpoint to fetch last run results of Amazon Search scraper.

14. **Add Set node "Edit Fields1" to clean Amazon product data:**  
    - Create `amazonURL` by concatenating "https://www.amazon.com" with product `dpUrl`.  
    - Extract `productTitle`, `price`, and shorten `productRating` to 3 characters.

15. **Add Filter node "Filter":**  
    - Condition: `productRating >= 4.3`.

16. **Add Limit node "Limit":**  
    - Limit output to 1 item.

17. **Add Set node "SaveItems":**  
    - Prepare fields `amazonURL`, `productTitle`, `price`, `productRating` for looping.

18. **Connect "SaveItems" output back to "Loop Over Items" to continue processing next Facebook ad item.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                         | Context or Link                                                                                                    |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------|
| This workflow requires valid API endpoints from Apify for the Facebook Ad Library scraper and Amazon Search scraper, configured under API > API Endpoints in n8n.                                                                  | Workflow setup prerequisites.                                                                                      |
| Optional Google Sheets integration can be used to store results; ensure you have a Google Sheets document ready and proper OAuth2 credentials configured in n8n.                                                                     | Data storage and reporting.                                                                                        |
| The workflow uses OpenAI GPT-4.1-mini model; ensure you have sufficient API quota and valid credentials set in n8n.                                                                                                                 | AI product name extraction.                                                                                        |
| The HTML to Plain Text conversion JS code removes scripts/styles, decodes entities, and formats text for cleaner AI input.                                                                                                         | Data preparation for AI input.                                                                                     |
| Email contact for workflow customizations or support: richard@advetica-systems.com                                                                                                                                                   | Author contact.                                                                                                    |
| ![Workflow Visual](https://i.imgur.com/fC1ppRW.png#full-width)                                                                                                                                                                      | Visual layout reference.                                                                                           |
| Sticky Notes within the workflow provide contextual guidance on API setup, node usage, and optional features; refer to them when editing or extending the workflow.                                                                  | Inline documentation.                                                                                              |

---

**Disclaimer:** The provided text and workflow originate exclusively from an automated n8n workflow. All data handled is legal and public, with strict adherence to content policies.