Google Maps Lead Generation with Apify & Email Extraction for Airtable

https://n8nworkflows.xyz/workflows/google-maps-lead-generation-with-apify---email-extraction-for-airtable-6307


# Google Maps Lead Generation with Apify & Email Extraction for Airtable

---

### 1. Workflow Overview

This workflow automates lead generation by scraping business information from Google Maps using an Apify actor, extracting website and contact details, particularly emails, and storing validated leads in Airtable. It is designed for marketing teams and sales professionals seeking to generate targeted business leads based on keyword and location inputs.

**Logical Blocks:**

- **1.1 Input Reception:** Captures user input from a web form specifying keywords, locations, and lead quantity.
- **1.2 Google Maps Data Extraction:** Runs an Apify actor to scrape business data from Google Maps and retrieves the dataset.
- **1.3 Data Filtering & Field Selection:** Limits the number of results and extracts relevant fields like company name, category, website, phone, and ratings.
- **1.4 Website URL Cleaning & HTTP Scraping:** Cleans scraped website URLs, sets realistic HTTP headers, scrapes website HTML content returning markdown.
- **1.5 Email Extraction & Validation:** Extracts email addresses from scraped website content, filters leads that have valid emails.
- **1.6 Data Storage:** Upserts filtered leads into Airtable with proper field mapping and duplicate handling.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:** Receives user input via a web form to specify the lead generation parameters.
- **Nodes Involved:**  
  - On form submission  
  - Sticky Note (Form Submission info)

- **Node Details:**

  - **On form submission**  
    - Type: Form Trigger  
    - Configuration:  
      - Form Title: "Web Scraper"  
      - Fields: Keyword (required), Location (required), No. Of Leads (optional, default 10)  
      - Description: Explains the form scrapes company info from Google Maps  
    - Input: External HTTP form submission webhook  
    - Output: JSON with user inputs  
    - Failure cases: Missing required fields, webhook downtime  
    - Version-specific: v2.2 for formTrigger  

  - **Sticky Note**  
    - Visual aid to indicate this block is for lead input  
    - No functional role  

#### 1.2 Google Maps Data Extraction

- **Overview:** Executes an Apify actor configured for Google Maps scraping using provided inputs; retrieves the resulting dataset for further processing.
- **Nodes Involved:**  
  - Run an Actor (Apify)  
  - Get dataset items (Apify)  
  - Grab Desired Fields (Set)  
  - Limit (Limit)  
  - Sticky Note (Scrape Business Info)

- **Node Details:**

  - **Run an Actor**  
    - Type: Apify node  
    - Configuration:  
      - Actor ID: "nwua9Gu5YrADL7ZDj" (Google Maps Scraper)  
      - Timeout: 120s, Wait for finish: 20s  
      - Input JSON includes: language=en, locationQuery from form, max places from No. Of Leads (default 10), search string from Keyword, skipClosedPlaces=false, website: "withWebsite"  
    - Credentials: Apify API Key  
    - Output: Actor run metadata including defaultDatasetId  
    - Failure cases: API auth errors, timeout, invalid inputs  

  - **Get dataset items**  
    - Type: Apify node  
    - Configuration: Retrieves all items from dataset with ID from previous node  
    - Credentials: Apify API Key  
    - Input: dataset ID from previous node  
    - Output: Raw scraped business data  
    - Failure cases: Dataset unavailable, API limit  

  - **Grab Desired Fields**  
    - Type: Set node  
    - Configuration: Maps dataset fields to simplified output: Company Name, Category, Address, Website, Phone Number, Rating, Other Categories (array)  
    - Input: dataset JSON  
    - Output: Cleaned partial business data with relevant fields  
    - Failure cases: Missing fields, null values  

  - **Limit**  
    - Type: Limit node  
    - Configuration: Limits output items to max 30  
    - Input: business data array  
    - Output: Limited array for batch processing  

  - **Sticky Note**  
    - Indicates this block scrapes business info via Apify and extracts fields  

#### 1.3 Data Filtering & Field Selection

- **Overview:** Processes each company data item in batches, preparing data for website scraping and further enrichment.
- **Nodes Involved:**  
  - Loop Over Items (splitInBatches)  
  - Parse url/website (Set)  
  - Remove Query Parameters & Fragments (Code)  
  - User-Agents (Set)  

- **Node Details:**

  - **Loop Over Items**  
    - Type: splitInBatches  
    - Configuration: Splits items to process sequentially or in batches  
    - Input: Limited business data  
    - Output: Single items for further nodes  
    - Failure cases: Empty input  

  - **Parse url/website**  
    - Type: Set node  
    - Configuration: Sets website URL field explicitly to prepare for cleaning  
    - Input: Single company data item  
    - Output: JSON with "website" attribute  

  - **Remove Query Parameters & Fragments**  
    - Type: Code node (JavaScript)  
    - Configuration:  
      - Cleans website URLs by:  
        - Prepending "https://" if missing scheme  
        - Parsing URL to remove query parameters and fragments  
      - Outputs cleanedWebsite, websiteType ("cleaned" or "invalid"), and error message if invalid  
    - Input: JSON with raw website URL  
    - Output: JSON with cleaned URL and tags  
    - Failure cases: Invalid URL format, empty strings  

  - **User-Agents**  
    - Type: Set node  
    - Configuration:  
      - Assigns a random User-Agent string from a predefined list simulating different browsers/devices  
      - Sets HTTP headers: Accept, Accept-Language, Referer, and User-Agent  
      - Carries forward the cleaned website URL for scraping  
    - Input: Cleaned URL JSON  
    - Output: JSON with scraping headers and website  

#### 1.4 Website URL Cleaning & HTTP Scraping

- **Overview:** Uses the cleaned website URLs and randomized HTTP headers to scrape the website content, outputting HTML in markdown format.
- **Nodes Involved:**  
  - Website Scraping (HTTP Request)  
  - Markdown (Markdown)  
  - Random Wait (Wait)  
  - Merge (Merge)  

- **Node Details:**

  - **Website Scraping**  
    - Type: HTTP Request  
    - Configuration:  
      - URL: from cleaned website field  
      - Sends headers set in User-Agents node  
      - Handles errors by continuing workflow (important for unreachable sites)  
    - Input: JSON with URL and headers  
    - Output: Raw HTML content in `data` field  
    - Failure cases: Request timeout, 404, blocked requests, invalid URLs  

  - **Markdown**  
    - Type: Markdown node  
    - Configuration: Converts raw HTML data into markdown format  
    - Input: HTTP response HTML  
    - Output: Markdown text in `data` field  
    - Failure cases: Malformed HTML, transformation errors  

  - **Random Wait**  
    - Type: Wait node  
    - Configuration: Waits random time between 2 and 7 seconds to avoid scraping detection/blocking  
    - Input: Markdown data  
    - Output: Delay before proceeding  

  - **Merge**  
    - Type: Merge node  
    - Configuration: Combines data streams by position from multiple parallel scraping threads  
    - Input: Batches of scraped data  
    - Output: Combined data for email extraction  

#### 1.5 Email Extraction & Validation

- **Overview:** Extracts email addresses from website markdown content, filters valid emails, and prepares data for final storage.
- **Nodes Involved:**  
  - Loop Over Items1 (splitInBatches)  
  - Extract Email Address (Code)  
  - Filter Leads with Email only (Filter)  
  - Grab Desired Fields1 (Set)  
  - Merge1 (Merge)  
  - Wait1 (Wait)  

- **Node Details:**

  - **Loop Over Items1**  
    - Type: splitInBatches  
    - Configuration: Processes individual items to extract emails and filter  
    - Input: Combined markdown data  
    - Output: Single items for email extraction  

  - **Extract Email Address**  
    - Type: Code node (JavaScript)  
    - Configuration:  
      - Uses regex to find email addresses in markdown content  
      - Filters out emails that look like filenames (e.g., .png, .jpg, .js)  
      - Deduplicates emails  
      - Returns first valid email or "N/A" and list of all found emails  
    - Input: Markdown content JSON  
    - Output: JSON with `email` and `allEmails`  
    - Failure cases: No emails found, malformed content  

  - **Filter Leads with Email only**  
    - Type: Filter node  
    - Configuration: Passes only items where email is not "N/A"  
    - Input: Email extraction results  
    - Output: Valid leads with emails only  

  - **Grab Desired Fields1**  
    - Type: Set node  
    - Configuration: Combines original company info with extracted email into final record structure  
    - Input: Filtered leads  
    - Output: JSON with all fields including Email for Airtable  

  - **Merge1**  
    - Type: Merge node  
    - Configuration: Combines processed items by position for batch insertion  

  - **Wait1**  
    - Type: Wait node  
    - Configuration: Waits 1 second before writing to database to avoid rate limits  

#### 1.6 Data Storage

- **Overview:** Inserts or updates valid lead records into Airtable, mapping fields and avoiding duplicates by email.
- **Nodes Involved:**  
  - Database (Airtable)  

- **Node Details:**

  - **Database**  
    - Type: Airtable node  
    - Configuration:  
      - Base and table selected for lead storage  
      - Upsert operation keyed on Email field (to avoid duplicates)  
      - Maps fields: Email, Rating, Website, Category, Location, Phone Number, Business Name, Other Categories (joined as string)  
    - Credentials: Airtable Personal Access Token  
    - Input: Final cleaned lead JSON  
    - Output: Airtable upsert response  
    - Failure cases: API limits, auth errors, data validation errors  

---

### 3. Summary Table

| Node Name                      | Node Type              | Functional Role                                  | Input Node(s)                 | Output Node(s)                | Sticky Note                                                    |
|-------------------------------|------------------------|-------------------------------------------------|------------------------------|------------------------------|---------------------------------------------------------------|
| On form submission             | Form Trigger           | Capture user input for lead generation          | (Webhook Input)               | Run an Actor                 |  ## Form Submission (Lead Input)                              |
| Run an Actor                  | Apify                  | Run Google Maps scraping actor                   | On form submission            | Get dataset items            |  ## Scrape Business Info (via Apify) and Grab desired fields |
| Get dataset items              | Apify                  | Retrieve scraped business dataset                | Run an Actor                 | Grab Desired Fields          |  ## Scrape Business Info (via Apify) and Grab desired fields |
| Grab Desired Fields            | Set                    | Extract relevant business fields                  | Get dataset items            | Limit                       |  ## Scrape Business Info (via Apify) and Grab desired fields |
| Limit                         | Limit                  | Limit number of leads processed                    | Grab Desired Fields          | Loop Over Items             |                                                               |
| Loop Over Items               | SplitInBatches         | Batch processing of leads                          | Limit                       | Loop Over Items1, Parse url/website |  ## Scrape Websites and return markdown                      |
| Parse url/website             | Set                    | Prepare website URL field                          | Loop Over Items             | Remove Query Parameters & Fragments |  ## Scrape Websites and return markdown                      |
| Remove Query Parameters & Fragments | Code              | Clean URLs by removing query/fragments             | Parse url/website            | User-Agents                 |  ## Scrape Websites and return markdown                      |
| User-Agents                   | Set                    | Assign randomized HTTP headers                      | Remove Query Parameters & Fragments | Website Scraping           |  ## Scrape Websites and return markdown                      |
| Website Scraping              | HTTP Request           | Scrape website HTML content                         | User-Agents                 | Markdown                    |  ## Scrape Websites and return markdown                      |
| Markdown                     | Markdown                | Convert HTML to markdown                            | Website Scraping            | Random Wait                 |  ## Scrape Websites and return markdown                      |
| Random Wait                  | Wait                   | Random delay to avoid detection                     | Markdown                   | Merge                      |                                                               |
| Merge                        | Merge                  | Combine scraping results                            | Random Wait, Loop Over Items | Loop Over Items1            |                                                               |
| Loop Over Items1             | SplitInBatches         | Batch processing for email extraction               | Merge                      | Filter Leads with Email only, Extract Email Address |  ## Extract Emails                                           |
| Extract Email Address        | Code                   | Extract and validate emails from markdown          | Loop Over Items1            | Wait1                      |  ## Extract Emails                                           |
| Filter Leads with Email only | Filter                 | Filter leads to only those with valid emails       | Loop Over Items1            | Grab Desired Fields1        |  ## Filter, Clean, and Store                                |
| Grab Desired Fields1         | Set                    | Combine company info and email into final record   | Filter Leads with Email only | Database                   |  ## Filter, Clean, and Store                                |
| Merge1                       | Merge                  | Combine filtered leads for database insertion      | Extract Email Address        | Wait1                      |  ## Filter, Clean, and Store                                |
| Wait1                        | Wait                   | Wait 1 second before Airtable upsert                | Merge1                     | Database                   |  ## Filter, Clean, and Store                                |
| Database                     | Airtable                | Upsert leads into Airtable base                      | Grab Desired Fields1        |                              |  ## Filter, Clean, and Store                                |
| Sticky Note                  | Sticky Note             | Visual annotation (Form input block)                |                              |                              |  ## Form Submission (Lead Input)                              |
| Sticky Note1                 | Sticky Note             | Visual annotation (Apify scraping block)             |                              |                              |  ## Scrape Business Info (via Apify) and Grab desired fields |
| Sticky Note2                 | Sticky Note             | Visual annotation (Website scraping block)           |                              |                              |  ## Scrape Websites and return markdown                      |
| Sticky Note3                 | Sticky Note             | Visual annotation (Email extraction block)            |                              |                              |  ## Extract Emails                                           |
| Sticky Note4                 | Sticky Note             | Visual annotation (Filtering and storing block)       |                              |                              |  ## Filter, Clean, and Store                                |
| Sticky Note5                 | Sticky Note             | Visual annotation (Setup instructions)                 |                              |                              | ## 🛠 How to Set It Up… See details in section 5             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger Node ("On form submission")**  
   - Type: Form Trigger (v2.2)  
   - Configure webhook with form title: "Web Scraper"  
   - Add 3 fields:  
     - Keyword (required)  
     - Location (required)  
     - No. Of Leads (optional, placeholder "10")  
   - Description: "This scrapes website urls from Google Maps to get company information."  

2. **Add Apify Node ("Run an Actor")**  
   - Type: Apify (v1)  
   - Credential: Apify API Key  
   - Operation: Run actor  
   - Actor ID: "nwua9Gu5YrADL7ZDj" (Google Maps Scraper)  
   - Timeout: 120 seconds, Wait for finish: 20 seconds  
   - Input JSON:  
     ```json
     {
       "language": "en",
       "locationQuery": "={{ $json.Location }}",
       "maxCrawledPlacesPerSearch": "={{ $json['No. Of Leads'] ? Number($json['No. Of Leads']) : 10 }}",
       "searchStringsArray": ["={{ $json.Keyword }}"],
       "skipClosedPlaces": false,
       "website": "withWebsite"
     }
     ```  
   - Connect "On form submission" to this node.  

3. **Add Apify Node ("Get dataset items")**  
   - Type: Apify (v1)  
   - Credential: Apify API Key  
   - Operation: Get items from dataset  
   - Dataset ID: "={{ $json.defaultDatasetId }}" (from previous node)  
   - Connect "Run an Actor" to this node.  

4. **Add Set Node ("Grab Desired Fields")**  
   - Type: Set (v3.4)  
   - Extract and assign:  
     - Company Name = {{$json.title}}  
     - Company Category = {{$json.categoryName}}  
     - Address = {{$json.address}}  
     - Website = {{$json.website}}  
     - Phone Number = {{$json.phoneUnformatted}}  
     - Rating = {{$json.totalScore}}  
     - Other Categories = {{$json.categories}} (Array)  
   - Connect "Get dataset items" to this node.  

5. **Add Limit Node ("Limit")**  
   - Type: Limit (v1)  
   - Max Items: 30  
   - Connect "Grab Desired Fields" to this node.  

6. **Add SplitInBatches Node ("Loop Over Items")**  
   - Type: SplitInBatches (v3)  
   - No special parameters (default batch size)  
   - Connect "Limit" to this node.  

7. **Add Set Node ("Parse url/website")**  
   - Type: Set (v3.4)  
   - Assign "website" = {{$json.Website}}  
   - Connect "Loop Over Items" to this node.  

8. **Add Code Node ("Remove Query Parameters & Fragments")**  
   - Type: Code (v2)  
   - JavaScript code:  
     ```javascript
     return items.map(item => {
       let rawUrl = item.json.website?.trim() || '';
       if (!/^https?:\/\//i.test(rawUrl)) {
         rawUrl = 'https://' + rawUrl;
       }
       try {
         const parsed = new URL(rawUrl);
         parsed.search = '';
         parsed.hash = '';
         item.json.cleanedWebsite = parsed.toString();
         item.json.websiteType = 'cleaned';
       } catch (err) {
         item.json.cleanedWebsite = null;
         item.json.websiteType = 'invalid';
         item.json.error = 'Invalid or missing URL';
       }
       return item;
     });
     ```  
   - Connect "Parse url/website" to this node.  

9. **Add Set Node ("User-Agents")**  
   - Type: Set (v3.4)  
   - Assign fields:  
     - website = {{$json.cleanedWebsite}}  
     - User-Agent = Random choice from array of 5 user agent strings (using expression with Math.random)  
     - Accept = "text/html,*/*;q=0.8"  
     - Accept-Language = "en-US,en;q=0.9"  
     - Referer = "https://google.com/"  
   - Connect "Remove Query Parameters & Fragments" to this node.  

10. **Add HTTP Request Node ("Website Scraping")**  
    - Type: HTTP Request (v4.2)  
    - URL: {{$json.website}}  
    - Send headers: User-Agent, Accept-Language, Referer, Accept (from JSON fields)  
    - On error: Continue workflow  
    - Connect "User-Agents" to this node.  

11. **Add Markdown Node ("Markdown")**  
    - Type: Markdown (v1)  
    - Convert HTML content from HTTP Request to markdown  
    - Connect "Website Scraping" to this node.  

12. **Add Wait Node ("Random Wait")**  
    - Type: Wait (v1.1)  
    - Wait amount: Random 2-7 seconds (expression: Math.floor(Math.random() * 6 + 2))  
    - Connect "Markdown" to this node.  

13. **Add Merge Node ("Merge")**  
    - Type: Merge (v3.2)  
    - Mode: Combine by position  
    - Connect "Random Wait" and "Loop Over Items" (second output) to this node.  

14. **Add SplitInBatches Node ("Loop Over Items1")**  
    - Type: SplitInBatches (v3)  
    - Connect "Merge" to this node.  

15. **Add Code Node ("Extract Email Address")**  
    - Type: Code (v2)  
    - JavaScript code:  
      ```javascript
      const emailRegex = /[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-z]{2,}/gi;
      let allEmails = [];
      for (const item of items) {
        const markdown = item.json.data || '';
        const matches = markdown.match(emailRegex);
        if (matches && matches.length) {
          allEmails.push(...matches);
        }
      }
      const filteredEmails = allEmails.filter(email => !email.toLowerCase().match(/\.(png|jpg|jpeg|gif|css|js)$/));
      const uniqueEmails = [...new Set(filteredEmails)];
      return [{
        json: {
          email: uniqueEmails[0] || "N/A",
          allEmails: uniqueEmails.length ? uniqueEmails : ["N/A"]
        }
      }];
      ```  
    - On error: Continue workflow  
    - Connect "Loop Over Items1" to this node.  

16. **Add Filter Node ("Filter Leads with Email only")**  
    - Type: Filter (v2.2)  
    - Condition: email != "N/A" (case sensitive)  
    - Connect "Loop Over Items1" (second output) to this node (for filtered)  
    - Connect "Extract Email Address" (second output) to "Merge1"  

17. **Add Set Node ("Grab Desired Fields1")**  
    - Type: Set (v3.4)  
    - Assign fields: Company Name, Company Category, Address, Website, Phone Number, Rating, Email, Other Categories (join array to string)  
    - Connect "Filter Leads with Email only" to this node.  

18. **Add Merge Node ("Merge1")**  
    - Type: Merge (v3.2)  
    - Mode: Combine by position  
    - Connect "Grab Desired Fields1" and "Extract Email Address" (second output) to this node.  

19. **Add Wait Node ("Wait1")**  
    - Type: Wait (v1.1)  
    - Wait amount: 1 second  
    - Connect "Merge1" to this node.  

20. **Add Airtable Node ("Database")**  
    - Type: Airtable (v2.1)  
    - Credentials: Airtable Personal Access Token  
    - Base: Select your Airtable base (e.g., "appBnug5XIRAWl5sK")  
    - Table: Select your Airtable table (e.g., "tblP5rgq3s76pYsCf")  
    - Operation: Upsert  
    - Match on column: Email  
    - Map fields: Email, Rating, Website, Category, Location, Phone Number, Business Name, Other Categories (joined string)  
    - Connect "Wait1" to this node.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                         | Context or Link                                                    |
|--------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------|
| 🛠 How to Set It Up: Open n8n, import this workflow JSON, set credentials for Apify and Airtable. Prepare Airtable base with columns: Email, Website, Phone, Company Name, etc. Review "Grab Desired Fields" nodes for customization. | Setup instructions from Sticky Note5                             |
| This workflow uses the Apify Google Maps Scraper actor: https://console.apify.com/actors/nwua9Gu5YrADL7ZDj                           | Actor reference                                                  |
| Uses random User-Agent headers to reduce scraping detection                                                                            | See "User-Agents" node details                                   |
| Email extraction filters out false positives ending with image/script extensions                                                       | See "Extract Email Address" node JavaScript                      |

---

**Disclaimer:** The provided text is derived solely from an automated workflow created with n8n, adhering strictly to current content policies and containing no illegal, offensive, or protected elements. All data processed is public and lawful.

---