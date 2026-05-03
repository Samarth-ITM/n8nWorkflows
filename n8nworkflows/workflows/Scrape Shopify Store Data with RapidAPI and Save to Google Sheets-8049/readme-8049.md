Scrape Shopify Store Data with RapidAPI and Save to Google Sheets

https://n8nworkflows.xyz/workflows/scrape-shopify-store-data-with-rapidapi-and-save-to-google-sheets-8049


# Scrape Shopify Store Data with RapidAPI and Save to Google Sheets

### 1. Workflow Overview

This n8n workflow automates the scraping of Shopify store data and product details using the Shopify Scraper API available via RapidAPI. It is designed for users to input a Shopify store website URL through a form submission, which triggers API requests to fetch comprehensive store information and product listings. The retrieved data is then appended into two separate Google Sheets for organized storage and later analysis.

The workflow logically divides into the following blocks:

- **1.1 Input Reception**: Captures the Shopify store URL submitted by the user via an n8n form trigger.
- **1.2 Store Information Retrieval**: Sends the store URL to a RapidAPI endpoint to obtain store-level metadata.
- **1.3 Product Data Retrieval**: Sends the same store URL to another RapidAPI endpoint to fetch product details.
- **1.4 Data Storage**: Appends the received store information and product data into designated Google Sheets tabs.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block initiates the workflow upon user input. It captures the Shopify store website URL from a form submission, which serves as the primary input for subsequent API requests.

- **Nodes Involved:**  
  - On form submission

- **Node Details:**  

  - **On form submission**  
    - Type: Form Trigger  
    - Role: Starts the workflow when a user submits the Shopify store URL via a form.  
    - Configuration:  
      - Form Title: "Shopify Scraper"  
      - Form Description: "Shopify Scraper"  
      - One field named `website` (text input expected to be a URL)  
    - Key Expressions: The form input is accessible via the JSON key `$json.website`.  
    - Input Connections: None (trigger node)  
    - Output Connections: Connects to both "Store Info Scrap Request" and "Products Scarp Request" nodes in parallel.  
    - Edge Cases/Potential Failures:  
      - Missing or invalid URL input by user may lead to downstream API errors.  
      - Form submission timeout or webhook failures.  
    - Version Requirements: Supports n8n v0.154+ with Form Trigger v2.2  

---

#### 2.2 Store Information Retrieval

- **Overview:**  
  Sends a POST request to the Shopify Scraper API endpoint to retrieve general store metadata, such as store name, location, currency, and description.

- **Nodes Involved:**  
  - Store Info Scrap Request

- **Node Details:**  

  - **Store Info Scrap Request**  
    - Type: HTTP Request  
    - Role: Sends store URL to RapidAPI endpoint `shopinfo.php` to fetch store information.  
    - Configuration:  
      - Method: POST  
      - URL: `https://shopify-scraper4.p.rapidapi.com/shopinfo.php`  
      - Headers:  
        - `x-rapidapi-host`: `shopify-scraper4.p.rapidapi.com`  
        - `x-rapidapi-key`: *User must supply their RapidAPI key here*  
      - Content-Type: `multipart-form-data`  
      - Body Parameter: `website` set dynamically to `{{$json.website}}` from the trigger node.  
      - Sends both body and headers with the request.  
    - Input Connections: From "On form submission"  
    - Output Connections: To "Append Store Info Google Sheets"  
    - Edge Cases/Potential Failures:  
      - API key missing or invalid leading to authentication errors.  
      - API rate limits or network timeouts.  
      - Invalid or unreachable website URL causing empty or error responses.  
      - Response format changes or malformed data.  
    - Version Requirements: HTTP Request v4.2+  

---

#### 2.3 Product Data Retrieval

- **Overview:**  
  Sends a POST request to another Shopify Scraper API endpoint to retrieve detailed product information from the Shopify store.

- **Nodes Involved:**  
  - Products Scarp Request

- **Node Details:**  

  - **Products Scarp Request**  
    - Type: HTTP Request  
    - Role: Sends store URL to RapidAPI endpoint `products.php` to fetch product listings and details.  
    - Configuration:  
      - Method: POST  
      - URL: `https://shopify-scraper4.p.rapidapi.com/products.php`  
      - Headers:  
        - `x-rapidapi-host`: `shopify-scraper4.p.rapidapi.com`  
        - `x-rapidapi-key`: *User must supply their RapidAPI key here*  
      - Content-Type: `multipart-form-data`  
      - Body Parameter: `website` dynamically set to `{{$json.website}}`.  
      - Sends both body and headers with the request.  
    - Input Connections: From "On form submission"  
    - Output Connections: To "Append Products Data In Google Sheets"  
    - Edge Cases/Potential Failures:  
      - Same as Store Info Scrap Request regarding API key, rate limits, network, and invalid URLs.  
      - Large product catalogs may cause longer response times or payload size issues.  
      - API changes affecting response structure.  
    - Version Requirements: HTTP Request v4.2+  

---

#### 2.4 Data Storage

- **Overview:**  
  Two Google Sheets nodes append the retrieved store and product data into separate sheets within a Google Spreadsheet document using a service account for authentication.

- **Nodes Involved:**  
  - Append Store Info Google Sheets  
  - Append Products Data In Google Sheets

- **Node Details:**  

  - **Append Store Info Google Sheets**  
    - Type: Google Sheets  
    - Role: Appends store metadata to the "Shop Info" sheet.  
    - Configuration:  
      - Operation: Append  
      - Document ID: Pre-configured Google Spreadsheet ID (not explicitly shown but must be set).  
      - Sheet Name: "Shop Info" (sheet ID `gid=0`)  
      - Column Mapping: Auto-maps fields such as id, name, city, province, country, currency, domain, description, etc.  
      - Authentication: Service Account (Google API credentials configured)  
      - Attempt to Convert Types: Disabled  
      - Convert Fields to String: Disabled  
    - Input Connections: From "Store Info Scrap Request"  
    - Output Connections: None  
    - Edge Cases/Potential Failures:  
      - Google API authentication failures if credentials expire or are revoked.  
      - Sheet ID or Document ID misconfiguration.  
      - Data schema mismatch causing append errors.  
      - API quota limits on Google Sheets.  
    - Version Requirements: Google Sheets node v4.6+  

  - **Append Products Data In Google Sheets**  
    - Type: Google Sheets  
    - Role: Appends product details to the "Products" sheet.  
    - Configuration:  
      - Operation: Append  
      - Document ID: Same Google Spreadsheet as store info.  
      - Sheet Name: "Products" (sheet ID 265041704)  
      - Column Mapping: Auto-maps fields like id, title, handle, body_html, published_at, vendor, product_type, tags, variants, images, options, etc.  
      - Authentication: Service Account (Google API credentials configured)  
      - Attempt to Convert Types: Disabled  
      - Convert Fields to String: Disabled  
    - Input Connections: From "Products Scarp Request"  
    - Output Connections: None  
    - Edge Cases/Potential Failures:  
      - Same as Append Store Info Google Sheets node.  
      - Large product datasets may hit API limits or cause performance delays.  
    - Version Requirements: Google Sheets node v4.6+  

---

### 3. Summary Table

| Node Name                      | Node Type           | Functional Role                                       | Input Node(s)          | Output Node(s)                             | Sticky Note                                                                                   |
|-------------------------------|---------------------|-----------------------------------------------------|------------------------|--------------------------------------------|-----------------------------------------------------------------------------------------------|
| On form submission             | Form Trigger        | Collects Shopify store URL input from user          | None                   | Store Info Scrap Request, Products Scarp Request | **On form submission** - Triggers the workflow when a user submits a website via form.        |
| Store Info Scrap Request       | HTTP Request        | Fetches Shopify store metadata via RapidAPI         | On form submission     | Append Store Info Google Sheets            | **Store Info Scrap Request** - Sends a POST request to fetch store-level metadata.            |
| Products Scarp Request         | HTTP Request        | Fetches product data from Shopify store via RapidAPI| On form submission     | Append Products Data In Google Sheets      | **Products Scarp Request** - Sends a POST request to retrieve product details.                |
| Append Store Info Google Sheets| Google Sheets       | Appends store info into "Shop Info" sheet            | Store Info Scrap Request| None                                       | **Append Store Info Google Sheets** - Appends store info into the "Shop Info" sheet.          |
| Append Products Data In Google Sheets | Google Sheets | Appends product data into "Products" sheet           | Products Scarp Request  | None                                       | **Append Products Data In Google Sheets** - Appends scraped product data into the sheet.      |
| Sticky Note                   | Sticky Note         | Workflow overview and summary                         | None                   | None                                       | Contains comprehensive workflow description and summary table.                                |
| Sticky Note1                  | Sticky Note         | Describes form submission node                        | None                   | None                                       | **On form submission** - Triggers the workflow when a user submits a website via form.        |
| Sticky Note2                  | Sticky Note         | Describes Store Info Scrap Request                    | None                   | None                                       | **Store Info Scrap Request** - Sends a POST request to fetch store-level metadata.            |
| Sticky Note3                  | Sticky Note         | Describes Products Scarp Request                       | None                   | None                                       | **Products Scarp Request** - Sends a POST request to retrieve product details.                |
| Sticky Note4                  | Sticky Note         | Describes Append Store Info Google Sheets             | None                   | None                                       | **Append Store Info Google Sheets** - Appends store info into the "Shop Info" sheet.          |
| Sticky Note5                  | Sticky Note         | Describes Append Products Data In Google Sheets       | None                   | None                                       | **Append Products Data In Google Sheets** - Appends scraped product data into the sheet.      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new n8n workflow.**

2. **Add a Form Trigger node: "On form submission"**  
   - Set Form Title to "Shopify Scraper".  
   - Set Form Description to "Shopify Scraper".  
   - Add one form field:  
     - Field Label: `website` (text input for Shopify store URL).  
   - This node will trigger the workflow when a user submits a URL.

3. **Add HTTP Request node: "Store Info Scrap Request"**  
   - Connect input from "On form submission".  
   - Set Method to POST.  
   - Set URL to `https://shopify-scraper4.p.rapidapi.com/shopinfo.php`.  
   - Under Headers, add:  
     - `x-rapidapi-host` with value `shopify-scraper4.p.rapidapi.com`.  
     - `x-rapidapi-key` with your personal RapidAPI key (must be obtained and kept secure).  
   - Set Content-Type to `multipart-form-data`.  
   - Under Body Parameters, add parameter:  
     - Name: `website`  
     - Value: `{{$json["website"]}}` (expression to use submitted URL).  
   - Enable sending both body and headers.

4. **Add HTTP Request node: "Products Scarp Request"**  
   - Connect input from "On form submission" (parallel to previous HTTP Request node).  
   - Set Method to POST.  
   - Set URL to `https://shopify-scraper4.p.rapidapi.com/products.php`.  
   - Headers same as "Store Info Scrap Request":  
     - `x-rapidapi-host`: `shopify-scraper4.p.rapidapi.com`  
     - `x-rapidapi-key`: your RapidAPI key.  
   - Content-Type: `multipart-form-data`.  
   - Body Parameters:  
     - `website` set to `{{$json["website"]}}`.  
   - Enable sending both body and headers.

5. **Add Google Sheets node: "Append Store Info Google Sheets"**  
   - Connect input from "Store Info Scrap Request".  
   - Operation: Append.  
   - Document ID: Enter your target Google Spreadsheet ID.  
   - Sheet Name: Set to "Shop Info" (or corresponding sheet tab).  
   - Authentication: Use Service Account credentials for Google API.  
   - Columns: Auto-map fields expected from store info JSON (id, name, city, province, country, currency, domain, description, etc.).  
   - Disable type conversion and string conversion.

6. **Add Google Sheets node: "Append Products Data In Google Sheets"**  
   - Connect input from "Products Scarp Request".  
   - Operation: Append.  
   - Document ID: Same as above.  
   - Sheet Name: Set to "Products".  
   - Authentication: Same service account credentials.  
   - Columns: Auto-map product data fields (id, title, handle, body_html, published_at, vendor, product_type, tags, variants, images, options, etc.).  
   - Disable type conversion and string conversion.

7. **Credential Setup:**  
   - Configure Google API credentials as a Service Account with permission to edit the target Google Spreadsheet.  
   - Store your RapidAPI key securely and reference it in the HTTP Request nodes.

8. **Workflow Connections:**  
   - "On form submission" node connects to both "Store Info Scrap Request" and "Products Scarp Request" in parallel.  
   - "Store Info Scrap Request" connects to "Append Store Info Google Sheets".  
   - "Products Scarp Request" connects to "Append Products Data In Google Sheets".

9. **Testing and Validation:**  
   - Test the form submission with a valid Shopify store URL.  
   - Ensure API requests return valid data without errors.  
   - Verify data is appended correctly to the respective Google Sheets tabs.

---

### 5. General Notes & Resources

| Note Content                                                                                                                     | Context or Link                                                   |
|----------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------|
| This workflow uses RapidAPI’s Shopify Scraper endpoints: `shopinfo.php` for store metadata and `products.php` for product data. | RapidAPI Shopify Scraper API documentation (user must obtain API key). |
| Google Sheets document URL example used in sticky notes: [Shopify Scraper Google Sheet](https://docs.google.com/spreadsheets/d/1mVC_2w7vHsKtcxUGkLWvzWSUqpzCoUWtWkLy3CPnfa4) | Example target spreadsheet for storing results.                  |
| Use service account credentials for Google API authentication to allow seamless append operations without user login prompts.  | Google Cloud Console and n8n documentation on service account setup. |
| Be aware of API rate limits on both RapidAPI and Google Sheets to prevent request throttling.                                   | General API usage best practices.                                |

---

**Disclaimer:**  
The provided description and analysis come exclusively from an n8n automated workflow. All data handled is legal and public, respecting content policies and containing no illegal or offensive material.