Extract Business Contact Info from Google Maps to Google Sheets with Regex

https://n8nworkflows.xyz/workflows/extract-business-contact-info-from-google-maps-to-google-sheets-with-regex-7943


# Extract Business Contact Info from Google Maps to Google Sheets with Regex

---

### 1. Workflow Overview

This workflow automates the extraction of business contact information from Google Maps search results and stores the data into a Google Sheets document for lead generation purposes. It is designed to help users compile structured business listings such as names, phone numbers, addresses, websites, and emails from a specified location and business type.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Search URL Construction:** Captures user-defined search parameters (business type and location) and constructs a Google Maps search URL accordingly.

- **1.2 Data Scraping and Extraction:** Performs an HTTP request to fetch Google Maps search results as raw HTML and uses regular expressions to extract business details.

- **1.3 Data Persistence:** Appends the extracted and structured business contact information into a predefined Google Sheets spreadsheet.

- **1.4 Workflow Trigger:** Manual start node to initiate the process.

- **1.5 Documentation Notes:** Sticky notes providing contextual explanations for each step.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Search URL Construction

- **Overview:**  
This block collects input parameters for the lead search—specifically the business type and location—and builds a Google Maps URL that will be used for scraping business listings.

- **Nodes Involved:**  
  - When clicking ‘Test workflow’  
  - Set Form Fields  
  - Build Search URL

- **Node Details:**

  - **When clicking ‘Test workflow’**  
    - *Type:* Manual Trigger  
    - *Role:* Initiates the workflow manually on user command.  
    - *Configuration:* Default manual trigger with no parameters.  
    - *Connections:* Outputs to "Set Form Fields".  
    - *Edge Cases:* None significant; only manual start.

  - **Set Form Fields**  
    - *Type:* Set Node  
    - *Role:* Defines static input parameters for the search query, including maximum number of results, business type, and location.  
    - *Configuration:*  
      - `max_results` = 10 (number)  
      - `lead_type` = "Call centers" (string)  
      - `location` = "New York" (string)  
    - *Key Expressions:* Static values set in node parameters.  
    - *Connections:* Receives from the manual trigger; outputs to "Build Search URL".  
    - *Edge Cases:* If parameters are invalid or empty, the search URL construction will be incorrect.

  - **Build Search URL**  
    - *Type:* Code Node (JavaScript)  
    - *Role:* Constructs a sanitized Google Maps search URL by replacing spaces with `+` symbols in the business type and location. Also passes along the max results parameter.  
    - *Configuration:*  
      - Input variables: `$json.lead_type` and `$json.location`  
      - Outputs a JSON object with keys:  
        - `search_url`: string URL for Google Maps search e.g. `https://www.google.com/maps/search/Call+centers+in+New+York`  
        - `max_results`: integer  
    - *Key Expressions:*  
      ```js
      const lead = $json.lead_type.replace(/\s+/g, '+');
      const loc = $json.location.replace(/\s+/g, '+');
      return {
        json: {
          search_url: `https://www.google.com/maps/search/${lead}+in+${loc}`,
          max_results: $json.max_results
        }
      };
      ```  
    - *Connections:* Receives from "Set Form Fields"; outputs to "Scrape Google Maps1".  
    - *Edge Cases:* If input strings contain special characters, URL encoding may be required but is not implemented here, possibly causing malformed URLs.

---

#### 2.2 Data Scraping and Extraction

- **Overview:**  
This block fetches Google Maps search results HTML and extracts business contact info using regular expressions. Extracted data includes business names, phone numbers, addresses, websites, and emails.

- **Nodes Involved:**  
  - Scrape Google Maps1  
  - Extract Business Info

- **Node Details:**

  - **Scrape Google Maps1**  
    - *Type:* HTTP Request  
    - *Role:* Performs an HTTP GET request to the constructed Google Maps search URL to retrieve the raw HTML content of the search results page.  
    - *Configuration:*  
      - URL: `={{ $json.search_url }}` (dynamic from previous node)  
      - Response options: Full HTTP response requested (`fullResponse: true`) to access raw HTML in the response body.  
    - *Connections:* Receives from "Build Search URL"; outputs to "Extract Business Info".  
    - *Edge Cases:*  
      - Google may block scraping or require CAPTCHA, resulting in HTTP 403 or 429 errors.  
      - Response format may change, invalidating regex extraction.  
      - Network timeouts or failures.

  - **Extract Business Info**  
    - *Type:* Code Node (JavaScript)  
    - *Role:* Parses the raw HTML string and applies multiple regex patterns to extract business names, phone numbers, addresses, websites, and emails. Aggregates results into JSON objects for each business.  
    - *Configuration:*  
      - Input: HTML content from HTTP request node  
      - Regex patterns used to extract:  
        - Business names from `aria-label` attributes  
        - Phone numbers in digit and symbol sequences  
        - Addresses from specific div classes  
        - Website URLs starting with http or https  
        - Emails excluding image file extensions  
      - Constructs an array of business info objects with keys: `name`, `phone`, `address`, `website`, `email`  
      - Uses fallback `'N/A'` for missing fields.  
    - *Key Expressions:*  
      ```js
      const html = $input.first().json.data;
      const businessRegex = /<div[^>]*aria-label="([^"]+)"/g;
      const phoneRegex = /(\+?\d[\d\s\-]{7,}\d)/g;
      const addressRegex = /<div class="[^"]*">([^<]*, [^<]*)<\/div>/g;
      const websiteRegex = /https?:\/\/[^\/\s"'>]+/g;
      const emailRegex = /[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.(?!jpeg|jpg|png|gif|webp|svg)[a-zA-Z]{2,}/g;

      // Extraction and result assembly follows...
      ```  
    - *Connections:* Receives from "Scrape Google Maps1"; outputs to "Save to Google Sheets".  
    - *Edge Cases:*  
      - Regex matches may be incomplete or inaccurate due to dynamic HTML structure changes.  
      - Missing data fields for some businesses.  
      - Multiple matches leading to misaligned arrays if lengths differ.  
      - Performance issues with large HTML data.

---

#### 2.3 Data Persistence

- **Overview:**  
This block takes the structured business contact data and appends it to a specified Google Sheets spreadsheet for easy access and further processing.

- **Nodes Involved:**  
  - Save to Google Sheets

- **Node Details:**

  - **Save to Google Sheets**  
    - *Type:* Google Sheets Node  
    - *Role:* Appends rows to a Google Sheets document with columns mapping to extracted business data fields.  
    - *Configuration:*  
      - Operation: Append  
      - Sheet name: "Sheet1"  
      - Document ID: Placeholder `"your_google_sheet_id_here"` to be replaced by actual Google Sheets document ID  
      - Column mappings:  
        - Email = `{{$json.email}}`  
        - Phone = `{{$json.phone}}`  
        - Address = `{{$json.address}}`  
        - Website = `{{$json.website}}`  
        - Business Name = `{{$json.name}}`  
      - Credentials: OAuth2 Google Sheets account configured externally.  
    - *Connections:* Receives from "Extract Business Info".  
    - *Edge Cases:*  
      - Authentication failures due to expired or invalid OAuth tokens.  
      - Incorrect document ID or sheet name causing append failures.  
      - Exceeding Google Sheets API quotas.  
      - Data format mismatches.

---

#### 2.4 Workflow Trigger

- **Overview:**  
Provides the manual start point for the entire workflow.

- **Nodes Involved:**  
  - When clicking ‘Test workflow’

- *Covered in section 2.1.*

---

#### 2.5 Documentation Notes

- **Overview:**  
Sticky notes provide stepwise explanations of the workflow’s logical sections for user clarity.

- **Nodes Involved:**  
  - Sticky Note (🗺️ STEP 1: Google Maps Data Extraction)  
  - Sticky Note1 (🔗 STEP 2: Website URL Processing)  
  - Sticky Note2 (📧 STEP 3: Email Extraction & Export)

- **Node Details:**

  - All sticky notes contain markdown-formatted content explaining the purpose and workflow logic for respective steps, helping users understand and modify the workflow.

---

### 3. Summary Table

| Node Name               | Node Type           | Functional Role                      | Input Node(s)                | Output Node(s)          | Sticky Note                                                                                      |
|-------------------------|---------------------|-----------------------------------|-----------------------------|-------------------------|-------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger      | Workflow start trigger             |                             | Set Form Fields          |                                                                                                |
| Set Form Fields          | Set Node            | Define search parameters           | When clicking ‘Test workflow’ | Build Search URL         |                                                                                                |
| Build Search URL         | Code Node           | Construct Google Maps search URL   | Set Form Fields              | Scrape Google Maps1      |                                                                                                |
| Scrape Google Maps1      | HTTP Request        | Fetch Google Maps search results   | Build Search URL             | Extract Business Info    |                                                                                                |
| Extract Business Info    | Code Node           | Extract business data via regex    | Scrape Google Maps1          | Save to Google Sheets    |                                                                                                |
| Save to Google Sheets    | Google Sheets       | Append extracted data to spreadsheet | Extract Business Info        |                         |                                                                                                |
| Sticky Note              | Sticky Note         | Explanation: Step 1 - Data Extraction |                             |                         | ## 🗺️ STEP 1: Google Maps Data Extraction<br>This workflow starts by scraping Google Maps...    |
| Sticky Note1             | Sticky Note         | Explanation: Step 2 - URL Processing |                             |                         | ## 🔗 STEP 2: Website URL Processing<br>Extracts and cleans business website URLs...             |
| Sticky Note2             | Sticky Note         | Explanation: Step 3 - Email Extraction & Export |                             |                         | ## 📧 STEP 3: Email Extraction & Export<br>Final processing pipeline including export to Sheets |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node**  
   - Name: `When clicking ‘Test workflow’`  
   - Purpose: To manually start the workflow.

2. **Add a Set node**  
   - Name: `Set Form Fields`  
   - Parameters:  
     - Add number field: `max_results` = 10  
     - Add string field: `lead_type` = "Call centers"  
     - Add string field: `location` = "New York"  
   - Connect output of Manual Trigger to this node.

3. **Add a Code node**  
   - Name: `Build Search URL`  
   - Language: JavaScript  
   - Code:  
     ```js
     const lead = $json.lead_type.replace(/\s+/g, '+');
     const loc = $json.location.replace(/\s+/g, '+');
     return {
       json: {
         search_url: `https://www.google.com/maps/search/${lead}+in+${loc}`,
         max_results: $json.max_results
       }
     };
     ```  
   - Connect output of `Set Form Fields` to this node.

4. **Add an HTTP Request node**  
   - Name: `Scrape Google Maps1`  
   - Method: GET  
   - URL: `={{ $json.search_url }}` (expression to use dynamic URL)  
   - Response Format: Leave default but enable option for full response if available to access raw HTML.  
   - Connect output of `Build Search URL` to this node.

5. **Add a Code node**  
   - Name: `Extract Business Info`  
   - Language: JavaScript  
   - Code:  
     ```js
     const html = $input.first().json.data;
     const results = [];

     const businessRegex = /<div[^>]*aria-label="([^"]+)"/g;
     const phoneRegex = /(\+?\d[\d\s\-]{7,}\d)/g;
     const addressRegex = /<div class="[^"]*">([^<]*, [^<]*)<\/div>/g;
     const websiteRegex = /https?:\/\/[^\/\s"'>]+/g;
     const emailRegex = /[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.(?!jpeg|jpg|png|gif|webp|svg)[a-zA-Z]{2,}/g;

     const names = [...html.matchAll(businessRegex)].map(m => m[1]);
     const phones = [...html.matchAll(phoneRegex)].map(m => m[1]);
     const addresses = [...html.matchAll(addressRegex)].map(m => m[1]);
     const websites = [...html.matchAll(websiteRegex)];
     const emails = [...html.matchAll(emailRegex)].map(m => m[0]);

     for (let i = 0; i < names.length; i++) {
       results.push({
         json: {
           name: names[i] || 'N/A',
           phone: phones[i] || 'N/A',
           address: addresses[i] || 'N/A',
           website: websites[i] ? websites[i][0] : 'N/A',
           email: emails[i] || 'N/A',
         }
       });
     }

     return results;
     ```  
   - Connect output of `Scrape Google Maps1` to this node.

6. **Add a Google Sheets node**  
   - Name: `Save to Google Sheets`  
   - Operation: Append  
   - Sheet Name: "Sheet1"  
   - Document ID: Replace `"your_google_sheet_id_here"` with your actual Google Sheets document ID.  
   - Columns mapping (define below):  
     - Email = `={{ $json.email }}`  
     - Phone = `={{ $json.phone }}`  
     - Address = `={{ $json.address }}`  
     - Website = `={{ $json.website }}`  
     - Business Name = `={{ $json.name }}`  
   - Credentials: Set up and select your Google Sheets OAuth2 credentials.  
   - Connect output of `Extract Business Info` to this node.

7. **Optional: Add Sticky Note nodes**  
   - Add three sticky note nodes to document each logical step with markdown content as in the original workflow for user reference.

8. **Verify all connections:**  
   - Manual Trigger → Set Form Fields → Build Search URL → Scrape Google Maps1 → Extract Business Info → Save to Google Sheets.

9. **Test the workflow:**  
   - Execute manually via the trigger node to validate the end-to-end process.

---

### 5. General Notes & Resources

| Note Content                                                                                                                      | Context or Link                                                                                  |
|-----------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| This workflow scrapes Google Maps directly by parsing HTML; no official Google Maps API is used, which may cause instability.     | Important for understanding potential scraping limitations and legal considerations.           |
| Replace the Google Sheets document ID placeholder with your own to enable data saving.                                            | Google Sheets integration requires appropriate OAuth2 credentials and correct document ID.     |
| Google may block scraping requests due to rate-limiting or anti-bot measures; consider adding delay or proxy rotation if needed. | Scraping best practices and mitigation techniques.                                             |
| Sticky notes in the workflow provide useful high-level guidance and explanation for each step.                                    | Use these notes to understand workflow logic quickly.                                          |

---

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

---