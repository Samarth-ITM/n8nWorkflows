Extract Business Emails from Google Maps to Google Sheets for Lead Generation

https://n8nworkflows.xyz/workflows/extract-business-emails-from-google-maps-to-google-sheets-for-lead-generation-7908


# Extract Business Emails from Google Maps to Google Sheets for Lead Generation

### 1. Workflow Overview

This workflow automates the extraction of business emails from Google Maps search results and exports them into a Google Sheets document for lead generation purposes. It is designed for agencies, sales teams, or marketers who want to gather business contact emails efficiently by leveraging Google Maps data combined with web scraping and filtering techniques.

The workflow is logically divided into four main blocks:

- **1.1 Google Maps Data Scraper:** Initiates the process by receiving a user input query (e.g., business type or location), performs a Google Maps search, and extracts website URLs from the search results.

- **1.2 URL Filtering & Processing:** Filters out irrelevant or Google-related URLs and removes duplicates to focus on valid business websites.

- **1.3 Smart Website Scraper:** Visits each filtered website URL, scrapes the page content, and extracts emails using regular expressions.

- **1.4 Email Extraction & Data Export:** Cleans extracted emails, removes duplicates, and appends the collected unique emails into a specified Google Sheets spreadsheet.

Additional informational sticky notes provide context and useful external resources.

---

### 2. Block-by-Block Analysis

#### 2.1 Google Maps Data Scraper

- **Overview:**  
Starts the workflow by listening for a chat input (search query), performs a Google Maps search with that query, and extracts URLs from the search results.

- **Nodes Involved:**  
  - When chat message received  
  - Scrape Google Maps  
  - Extract URLs  
  - Filter Google URLs  
  - Remove Duplicates  
  - Sticky Note (Step 1)

- **Node Details:**  
  - **When chat message received**  
    - *Type:* Chat Trigger (Langchain)  
    - *Role:* Entry point, triggers workflow when a chat message is received with user input (search query).  
    - *Key Config:* Listens for any chat message; assumes the message text is the search query.  
    - *Input:* External chat message event  
    - *Output:* Passes chat input to next node  
    - *Failure Modes:* Trigger might fail if webhook is not configured or chat service down.

  - **Scrape Google Maps**  
    - *Type:* HTTP Request  
    - *Role:* Performs a Google Maps search using the chat input query.  
    - *Config:* URL dynamically constructed as `https://www.google.com/maps/search/{{ $json.chatInput }}`  
    - *Output:* Full HTTP response, including HTML data containing business info and links.  
    - *Edge Cases:* Google Maps could block scraping requests, CAPTCHAs, or require authentication.

  - **Extract URLs**  
    - *Type:* Code (JavaScript)  
    - *Role:* Parses the HTML data from Google Maps search and extracts all URLs using regex.  
    - *Regex Used:* `/https?:\/\/[^\/\s"'>]+/g` — matches all HTTP/HTTPS URLs.  
    - *Output:* Array of objects with extracted website URLs.  
    - *Failure Modes:* If no URLs found, output is empty; regex may capture unwanted URLs.

  - **Filter Google URLs**  
    - *Type:* Filter  
    - *Role:* Removes URLs containing substrings related to Google services (like 'schema', 'google', 'gg', 'gstatic') to exclude Google-owned URLs.  
    - *Conditions:* Checks that `website` field does NOT contain any of those substrings.  
    - *Output:* Filtered list of URLs.  
    - *Edge Cases:* May incorrectly filter out some valid URLs if they contain these substrings.

  - **Remove Duplicates**  
    - *Type:* Remove Duplicates  
    - *Role:* Eliminates duplicate URLs to avoid redundant scraping downstream.  
    - *Output:* Unique URLs only.

  - **Sticky Note (Step 1)**  
    - Annotates this block as "🔎 Step 1: Google Maps Data Scraper".

---

#### 2.2 URL Filtering & Processing

- **Overview:**  
Handles further processing of filtered URLs by splitting them into manageable batches and preparing for scraping individual websites.

- **Nodes Involved:**  
  - Loop Over Items (splitInBatches)  
  - Wait1  
  - Sticky Note1 (Step 2)

- **Node Details:**  
  - **Loop Over Items**  
    - *Type:* Split In Batches  
    - *Role:* Processes URLs in batches to control request rates and avoid overloading target servers or triggering anti-scraping defenses.  
    - *Config:* Default batch size (unspecified), processes each batch sequentially.  
    - *Input:* Filtered unique URLs.  
    - *Output:* Batches of URLs for sequential processing.

  - **Wait1**  
    - *Type:* Wait  
    - *Role:* Delay between each batch processing to respect rate limits and avoid IP blocking.  
    - *Config:* Default wait time (unspecified, likely zero or minimal).  
    - *Output:* Continues processing after wait.

  - **Sticky Note1 (Step 2)**  
    - Labels this block as "🔗 Step 2: URL Filtering & Processing".

---

#### 2.3 Smart Website Scraper

- **Overview:**  
Scrapes each website from the filtered URLs batch, extracts page content, then extracts emails from that content.

- **Nodes Involved:**  
  - Scrape Site  
  - Wait  
  - Extract Emails  
  - Filter Out Empties  
  - Split Out  
  - Remove Duplicates (2)  
  - Sticky Note2 (Step 3)  
  - Sticky Note3 (Step 4)

- **Node Details:**  
  - **Scrape Site**  
    - *Type:* HTTP Request  
    - *Role:* Fetches the HTML content of each website URL for email extraction.  
    - *Config:* URL dynamically set to `{{$json.website}}` from batch output; follows redirects.  
    - *Error Handling:* On error, continues workflow without stopping (to handle inaccessible or blocked sites).  
    - *Output:* Website HTML content.

  - **Wait**  
    - *Type:* Wait  
    - *Role:* Introduces delay after each website scrape to avoid rate-limiting or blocking.  
    - *Config:* Wait time of 1 second.  

  - **Extract Emails**  
    - *Type:* Code (JavaScript)  
    - *Role:* Uses regex to extract emails from scraped website HTML content.  
    - *Regex Used:* `/[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.(?!jpeg|jpg|png|gif|webp|svg)[a-zA-Z]{2,}/g`  
      - This regex excludes image file extensions to avoid false positives.  
    - *Output:* JSON object containing an `emails` array.  
    - *OnError:* Continues output even if extraction fails (robustness).  

  - **Filter Out Empties**  
    - *Type:* Filter  
    - *Role:* Removes records where the extracted emails array does not exist or is empty.  
    - *Output:* Only items with actual emails.

  - **Split Out**  
    - *Type:* Split Out  
    - *Role:* Splits the `emails` array into individual email records for downstream processing.

  - **Remove Duplicates (2)**  
    - *Type:* Remove Duplicates  
    - *Role:* Removes any duplicate email addresses before export.  

  - **Sticky Note2 (Step 3)**  
    - Marks this block as "🔄 Step 3: Smart Website Scraper".

  - **Sticky Note3 (Step 4)**  
    - Marks the next block for email extraction and export as "📧 Step 4: Email Extraction & Data Export".

---

#### 2.4 Email Extraction & Data Export

- **Overview:**  
Finalizes the data by appending unique emails into a Google Sheets document for lead generation.

- **Nodes Involved:**  
  - Add to Sheet (or whatever you want!)  
  - Sticky Note3 (Step 4)  
  - Sticky Note5 (Resources)

- **Node Details:**  
  - **Add to Sheet (or whatever you want!)**  
    - *Type:* Google Sheets  
    - *Role:* Appends the unique emails into a predefined Google Sheets spreadsheet and sheet.  
    - *Config:*  
      - Document ID: `10V7ikaGWmC-U73Z0sWOv1hfwbyMDvYNMS0lxlvA6C8c`  
      - Sheet Name: `gid=0` (first sheet)  
      - Operation: Append  
      - Columns: Single column named `emails` mapped from JSON `emails` field.  
      - Matching columns: `emails` (to avoid duplicates if necessary)  
    - *Credentials:* Google Sheets OAuth2 account configured.  
    - *Failure Modes:* Authentication errors, API rate limits, spreadsheet permission issues.

  - **Sticky Note5 (Resources)**  
    - Provides external resource link to [Noman Mohammad](https://nomanmohammad.com) for advanced workflows and lead generation strategies.

---

### 3. Summary Table

| Node Name                    | Node Type               | Functional Role                          | Input Node(s)                    | Output Node(s)                   | Sticky Note                                                         |
|------------------------------|-------------------------|----------------------------------------|---------------------------------|---------------------------------|--------------------------------------------------------------------|
| When chat message received    | Chat Trigger (Langchain) | Entry trigger for user search input    | -                               | Scrape Google Maps              |                                                                    |
| Scrape Google Maps            | HTTP Request            | Performs Google Maps search             | When chat message received       | Extract URLs                   | 🔎 Step 1: Google Maps Data Scraper                                |
| Extract URLs                 | Code                    | Extracts URLs from Google Maps result   | Scrape Google Maps               | Filter Google URLs              | 🔎 Step 1: Google Maps Data Scraper                                |
| Filter Google URLs            | Filter                  | Filters out Google-owned URLs            | Extract URLs                    | Remove Duplicates               | 🔗 Step 2: URL Filtering & Processing                              |
| Remove Duplicates             | Remove Duplicates       | Removes duplicate URLs                   | Filter Google URLs               | Loop Over Items                | 🔗 Step 2: URL Filtering & Processing                              |
| Loop Over Items              | Split In Batches        | Batches URLs for controlled processing  | Remove Duplicates                | Wait1, Scrape Site             | 🔗 Step 2: URL Filtering & Processing                              |
| Wait1                       | Wait                    | Waits between batches                    | Loop Over Items                  | Filter Out Empties             | 🔗 Step 2: URL Filtering & Processing                              |
| Filter Out Empties            | Filter                  | Removes items without emails             | Wait1                          | Split Out                     | 📧 Step 4: Email Extraction & Data Export                         |
| Split Out                   | Split Out               | Splits emails array into individual items | Filter Out Empties              | Remove Duplicates (2)          | 📧 Step 4: Email Extraction & Data Export                         |
| Remove Duplicates (2)         | Remove Duplicates       | Removes duplicate emails                  | Split Out                      | Add to Sheet (or whatever you want!) | 📧 Step 4: Email Extraction & Data Export                         |
| Add to Sheet (or whatever you want!) | Google Sheets        | Appends unique emails to Google Sheets  | Remove Duplicates (2)            | -                               | 📧 Step 4: Email Extraction & Data Export                         |
| Scrape Site                  | HTTP Request            | Scrapes each website’s HTML content      | Loop Over Items                 | Wait                          | 🔄 Step 3: Smart Website Scraper                                  |
| Wait                        | Wait                    | Waits after each site scrape              | Scrape Site                    | Extract Emails                | 🔄 Step 3: Smart Website Scraper                                  |
| Extract Emails              | Code                    | Extracts emails via regex from site HTML | Wait                          | Loop Over Items               | 🔄 Step 3: Smart Website Scraper                                  |
| Sticky Note                  | Sticky Note             | Annotation for Step 1                     | -                              | -                             | ## 🔎 Step 1: Google Maps Data Scraper                            |
| Sticky Note1                 | Sticky Note             | Annotation for Step 2                     | -                              | -                             | ## 🔗 Step 2: URL Filtering & Processing                          |
| Sticky Note2                 | Sticky Note             | Annotation for Step 3                     | -                              | -                             | ## 🔄 Step 3: Smart Website Scraper                              |
| Sticky Note3                 | Sticky Note             | Annotation for Step 4                     | -                              | -                             | ## 📧 Step 4: Email Extraction & Data Export                     |
| Sticky Note5                 | Sticky Note             | External resources and advanced workflows | -                              | -                             | ## 🚀 Get More Resources & Advanced Workflows [Noman Mohammad](https://nomanmohammad.com) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Node: When chat message received**  
   - Type: `@n8n/n8n-nodes-langchain.chatTrigger`  
   - Configuration: Default, no special parameters needed; this triggers on any chat message.  
   - Connect output to **Scrape Google Maps** node.

2. **Create Node: Scrape Google Maps**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://www.google.com/maps/search/{{ $json.chatInput }}` (use expression to insert chat input)  
   - Response: Full response enabled  
   - Allow unauthorized certs: Enabled  
   - Connect output to **Extract URLs**.

3. **Create Node: Extract URLs**  
   - Type: Code (JavaScript)  
   - Code:
     ```js
     const input = $input.first().json.data;
     const regex = /https?:\/\/[^\/\s"'>]+/g;
     const websites = input.match(regex);
     return websites.map(website => ({json:{website}}));
     ```
   - Connect output to **Filter Google URLs**.

4. **Create Node: Filter Google URLs**  
   - Type: Filter  
   - Conditions:  
     - `website` NOT CONTAIN "schema"  
     - `website` NOT CONTAIN "google"  
     - `website` NOT CONTAIN "gg"  
     - `website` NOT CONTAIN "gstatic"  
   - Connect output to **Remove Duplicates**.

5. **Create Node: Remove Duplicates**  
   - Type: Remove Duplicates  
   - Default configuration (removes duplicates across all fields)  
   - Connect output to **Loop Over Items**.

6. **Create Node: Loop Over Items**  
   - Type: Split In Batches  
   - Batch size: Default or set to a sensible number (e.g., 5-10) to avoid rate limits.  
   - Connect outputs to two nodes: **Wait1** and **Scrape Site** (parallel processing).

7. **Create Node: Wait1**  
   - Type: Wait  
   - Default wait time (can be 0 or a few seconds if needed).  
   - Connect output to **Filter Out Empties**.

8. **Create Node: Scrape Site**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `{{$json.website}}` (expression from batch item)  
   - Follow redirects: Enabled  
   - On error: Continue workflow (do not fail)  
   - Connect output to **Wait**.

9. **Create Node: Wait**  
   - Type: Wait  
   - Wait time: 1 second (to avoid rate-limiting)  
   - Connect output to **Extract Emails**.

10. **Create Node: Extract Emails**  
    - Type: Code (JavaScript)  
    - Code:
      ```js
      const input = $input.first().json.data;
      const regex = /[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.(?!jpeg|jpg|png|gif|webp|svg)[a-zA-Z]{2,}/g;
      const emails = input.match(regex);
      return {json: {emails: emails}};
      ```
    - On error: Continue  
    - Connect output to **Loop Over Items** (to process next batch).

11. **Create Node: Filter Out Empties**  
    - Type: Filter  
    - Condition: `emails` field EXISTS (array with values)  
    - Connect output to **Split Out**.

12. **Create Node: Split Out**  
    - Type: Split Out  
    - Field to split out: `emails`  
    - Connect output to **Remove Duplicates (2)**.

13. **Create Node: Remove Duplicates (2)**  
    - Type: Remove Duplicates  
    - Default configuration to remove duplicate emails.  
    - Connect output to **Add to Sheet (or whatever you want!)**.

14. **Create Node: Add to Sheet (or whatever you want!)**  
    - Type: Google Sheets  
    - Operation: Append  
    - Document ID: `10V7ikaGWmC-U73Z0sWOv1hfwbyMDvYNMS0lxlvA6C8c`  
    - Sheet Name: `gid=0`  
    - Columns: Define one column `emails` mapped from JSON `emails` field  
    - Matching Columns: `emails` (to prevent duplicates if re-imported)  
    - Credentials: Configure Google Sheets OAuth2 credentials with access to target spreadsheet.

15. **Add Sticky Notes** (optional but recommended for clarity):  
    - Step 1: Near When chat message received and Scrape Google Maps  
    - Step 2: Near Filter Google URLs and Loop Over Items  
    - Step 3: Near Scrape Site, Wait, Extract Emails  
    - Step 4: Near Filter Out Empties, Split Out, Remove Duplicates (2), Add to Sheet  
    - Resources note linking to [Noman Mohammad](https://nomanmohammad.com)

---

### 5. General Notes & Resources

| Note Content                                                                                                                        | Context or Link                                                      |
|------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------|
| For additional resources, advanced automation tutorials, and business strategies that help generate more leads and grow your agency, visit: [Noman Mohammad](https://nomanmohammad.com) | Sticky Note5 in workflow, external resource for lead generation    |
| The workflow uses regex patterns carefully tailored to avoid extracting image file extensions as emails, improving accuracy.       | General workflow design note                                        |
| Google Maps scraping may be subject to anti-scraping mechanisms (CAPTCHAs, IP blocks). Use proxies or APIs if scaling is needed.    | Operational consideration                                          |
| Google Sheets OAuth2 credential must have write access to the target spreadsheet to append data successfully.                       | Credential configuration note                                       |
| The workflow handles errors gracefully in web scraping steps, ensuring that failures at individual sites do not stop the entire process. | Robustness and error handling best practice                        |

---

**Disclaimer:** The provided text is generated exclusively based on an automated n8n workflow. It strictly complies with content policies and contains no illegal or protected elements. All data processed is legal and publicly accessible.