Web Scraper: Extract Website Content from Sitemaps to Google Drive

https://n8nworkflows.xyz/workflows/web-scraper--extract-website-content-from-sitemaps-to-google-drive-7186


# Web Scraper: Extract Website Content from Sitemaps to Google Drive

### 1. Workflow Overview

This workflow, titled **"Web Scraper: Extract Website Content from Sitemaps to Google Drive"**, is designed to fully automate the process of scraping website content by leveraging sitemap XML files. It targets use cases where systematic extraction of multiple pages from a website is needed, such as content archiving, SEO auditing, or data analysis.

The workflow operates in clearly defined logical blocks:

- **1.1 Input Reception and Sitemap URL Setup:** Manual trigger initiates the workflow and sets the target sitemap URL.
- **1.2 Sitemap Retrieval and Parsing:** Retrieves the sitemap XML via HTTP, parses it to extract all page URLs.
- **1.3 URL Processing Loop:** Optionally limits URLs, then loops through each URL sequentially.
- **1.4 Page Scraping and Content Extraction:** For each URL, performs an HTTP GET request to fetch the page HTML, then extracts structured content using custom JavaScript.
- **1.5 Content Formatting:** Formats the extracted data into a Markdown document with metadata.
- **1.6 Saving Output and Throttling:** Saves the formatted content as a Markdown file on Google Drive and waits between requests to respect rate limits and avoid server overload.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Sitemap URL Setup

- **Overview:** This initial block provides the manual trigger to start the workflow and sets the sitemap URL from which pages will be scraped.
- **Nodes Involved:**  
  - `Start Working Scraper` (Manual Trigger)  
  - `Set Sitemap URL` (Set Node)  
  - `Working Scraper Note` (Sticky Note)

- **Node Details:**

  - **Start Working Scraper**  
    - Type: Manual Trigger  
    - Role: Starts workflow execution manually on user command.  
    - Configuration: No parameters; simple trigger node.  
    - Inputs: None  
    - Outputs: Connects to `Set Sitemap URL`  
    - Failure Modes: None  

  - **Set Sitemap URL**  
    - Type: Set Node  
    - Role: Assigns the sitemap URL string for downstream use.  
    - Configuration: Sets a string variable `sitemap_url` to `"https://yourwebsitehere.com/page-sitemap.xml"`.  
    - Inputs: From `Start Working Scraper`  
    - Outputs: To `Get Sitemap`  
    - Edge Cases: Needs user to replace the placeholder URL with a valid sitemap URL.  
    - Notes: Critical for directing the scraping process.

  - **Working Scraper Note**  
    - Type: Sticky Note  
    - Role: Provides user-facing documentation and notes inside the workflow editor.  
    - Content: Highlights workflow features such as sitemap support, sequential processing, delays, and robust error handling.  
    - Inputs/Outputs: None  

#### 2.2 Sitemap Retrieval and Parsing

- **Overview:** Fetches the sitemap XML from the URL and parses it to extract URLs for scraping.
- **Nodes Involved:**  
  - `Get Sitemap` (HTTP Request)  
  - `Parse Sitemap XML` (XML Node)  
  - `Split Sitemap URLs` (Split Out Node)  

- **Node Details:**

  - **Get Sitemap**  
    - Type: HTTP Request  
    - Role: Downloads the sitemap XML content.  
    - Configuration:  
      - URL: Dynamic from `sitemap_url` variable.  
      - Timeout: 15 seconds to avoid hanging.  
      - Response configured to never error on HTTP failure codes (e.g., 404), allowing graceful fallback.  
    - Inputs: From `Set Sitemap URL`  
    - Outputs: To `Parse Sitemap XML`  
    - Edge Cases: Network errors, invalid sitemap URL, or sitemap not accessible.  

  - **Parse Sitemap XML**  
    - Type: XML Parsing Node  
    - Role: Parses the XML response into JSON for easier handling.  
    - Configuration: Default XML parsing with no special options.  
    - Inputs: From `Get Sitemap`  
    - Outputs: To `Split Sitemap URLs`  
    - Edge Cases: Malformed XML or unexpected sitemap structure may cause parsing errors.

  - **Split Sitemap URLs**  
    - Type: Split Out Node  
    - Role: Splits the parsed sitemap array (`urlset.url`) into individual items (single URLs) for processing.  
    - Configuration: Splits on field `urlset.url`  
    - Inputs: From `Parse Sitemap XML`  
    - Outputs: To `Limit URLs (Optional)`  
    - Edge Cases: Empty sitemap or missing `urlset.url` field.

#### 2.3 URL Processing Loop

- **Overview:** Optionally limits the number of URLs to process and feeds the URLs one-by-one into a batch loop for sequential scraping.
- **Nodes Involved:**  
  - `Limit URLs (Optional)` (Limit Node, Disabled)  
  - `Loop URLs` (Split In Batches Node)

- **Node Details:**

  - **Limit URLs (Optional)**  
    - Type: Limit Node  
    - Role: Restricts the number of URLs processed (e.g., to 20) for testing or resource control.  
    - Configuration: Max items set to 20, but the node is disabled by default.  
    - Inputs: From `Split Sitemap URLs`  
    - Outputs: To `Loop URLs`  
    - Notes: User can enable and adjust limit to control workflow scope.

  - **Loop URLs**  
    - Type: Split In Batches  
    - Role: Processes URLs sequentially in batches of size 1 (default), enabling rate control and error handling per URL.  
    - Inputs: From `Limit URLs (Optional)` (or directly from `Split Sitemap URLs` if limit disabled)  
    - Outputs: To `Scrape Page`  
    - Edge Cases: Large sitemaps may extend workflow runtime; batch size tuning possible.

#### 2.4 Page Scraping and Content Extraction

- **Overview:** For each URL, requests the HTML page, extracts meaningful content and metadata using custom JavaScript.
- **Nodes Involved:**  
  - `Scrape Page` (HTTP Request)  
  - `Extract Content` (Code Node)

- **Node Details:**

  - **Scrape Page**  
    - Type: HTTP Request  
    - Role: Fetches the full HTML content of the current URL.  
    - Configuration:  
      - URL dynamic: `{{$json.loc}}` where `loc` is the URL from sitemap.  
      - Timeout: 20 seconds.  
      - Custom HTTP headers mimicking a real browser (User-Agent, Accept, Cache-Control, etc.) to reduce blocking by anti-bot systems.  
      - Response set to never error on HTTP failure codes.  
    - Inputs: From `Loop URLs`  
    - Outputs: To `Extract Content`  
    - Edge Cases: HTTP errors, redirects, JavaScript-rendered content not visible in static HTML, anti-bot blocks.

  - **Extract Content**  
    - Type: Code Node (JavaScript)  
    - Role: Parses the HTML response safely, extracts title, readable textual content, and metadata such as word count and framework indicators.  
    - Configuration: Custom JS code that:  
      - Handles different response formats reliably.  
      - Extracts `<title>`, Open Graph title, or `<h1>` tags for page title.  
      - Cleans HTML by removing scripts, styles, comments, and converts to readable markdown-like text.  
      - Detects if the page uses Divi or WordPress frameworks or contains JavaScript.  
      - Returns a structured object with extracted data and flags indicating success and page characteristics.  
    - Inputs: From `Scrape Page`  
    - Outputs: To `Format Content`  
    - Edge Cases: Pages with heavy JavaScript content, anti-scraping measures, malformed HTML, or unexpected page structures.

#### 2.5 Content Formatting

- **Overview:** Formats the extracted content and metadata into a Markdown document with frontmatter metadata, suitable for storage or further processing.
- **Nodes Involved:**  
  - `Format Content` (Code Node)

- **Node Details:**

  - **Format Content**  
    - Type: Code Node (JavaScript)  
    - Role: Converts the extracted structured content into a Markdown document with YAML frontmatter metadata.  
    - Configuration:  
      - Includes metadata such as title, URL, scrape timestamp, word count, HTML length, HTTP status, framework detections, and scraper type.  
      - Adds warnings if scraping failed or if content is limited due to JavaScript dependencies.  
      - Formats the main content with proper Markdown headers and sections.  
      - Provides technical details summary at the end.  
    - Inputs: From `Extract Content`  
    - Outputs: To `Save to Google Drive`  
    - Edge Cases: Handles scraping failures gracefully by indicating a failed scrape in output.

#### 2.6 Saving Output and Throttling

- **Overview:** Saves the formatted Markdown content as a file on Google Drive and enforces a 3-second delay between page scrapes to avoid server overload.
- **Nodes Involved:**  
  - `Save to Google Drive` (Google Drive Node)  
  - `Wait Between Pages` (Wait Node)

- **Node Details:**

  - **Save to Google Drive**  
    - Type: Google Drive Node  
    - Role: Creates a new text file containing the scraped page content on Google Drive.  
    - Configuration:  
      - Filename generated from URL by stripping protocol and replacing slashes with underscores, appended with `_sitemap.md`.  
      - Content from `Format Content` node’s `formatted` field.  
      - Target folder specified by Google Drive folder ID and name (placeholders to be configured by user).  
      - Uses OAuth2 credentials for Google Drive access (user must provide credentials).  
    - Inputs: From `Format Content`  
    - Outputs: To `Wait Between Pages`  
    - Edge Cases: Google API quota limits, credential expiration, folder access permissions.

  - **Wait Between Pages**  
    - Type: Wait Node  
    - Role: Pauses workflow for 3 seconds before processing next URL to be respectful to target servers.  
    - Configuration: Wait duration set to 3 seconds.  
    - Inputs: From `Save to Google Drive`  
    - Outputs: To `Loop URLs` (feeding the next batch)  
    - Edge Cases: Delays add to total runtime; network or node timeouts are possible if misconfigured.

---

### 3. Summary Table

| Node Name              | Node Type            | Functional Role                    | Input Node(s)           | Output Node(s)          | Sticky Note                                                                                                             |
|------------------------|----------------------|----------------------------------|------------------------|-------------------------|-------------------------------------------------------------------------------------------------------------------------|
| Working Scraper Note    | Sticky Note          | Documentation and workflow notes | None                   | None                    | ## ✅ Simple Working Scraper<br>### Now with Full Sitemap Support<br>**Uses direct HTTP requests that we know work**<br>**Features:**<br>- Fetches complete sitemap<br>- Processes all pages sequentially<br>- Respectful 3-second delays<br>- Robust error handling |
| Start Working Scraper   | Manual Trigger       | Starts the workflow manually      | None                   | Set Sitemap URL          |                                                                                                                         |
| Set Sitemap URL         | Set                  | Defines the sitemap URL           | Start Working Scraper   | Get Sitemap              |                                                                                                                         |
| Get Sitemap             | HTTP Request         | Downloads sitemap XML             | Set Sitemap URL         | Parse Sitemap XML        |                                                                                                                         |
| Parse Sitemap XML       | XML                   | Parses XML to JSON                | Get Sitemap             | Split Sitemap URLs       |                                                                                                                         |
| Split Sitemap URLs      | Split Out             | Splits sitemap into URL items    | Parse Sitemap XML       | Limit URLs (Optional)    |                                                                                                                         |
| Limit URLs (Optional)   | Limit (Disabled)      | Optional URL count limiter        | Split Sitemap URLs      | Loop URLs                |                                                                                                                         |
| Loop URLs              | Split In Batches      | Processes URLs sequentially       | Limit URLs (Optional)   | Scrape Page              |                                                                                                                         |
| Scrape Page             | HTTP Request         | Fetches HTML content of each URL | Loop URLs               | Extract Content          |                                                                                                                         |
| Extract Content         | Code (JavaScript)     | Extracts and cleans page content | Scrape Page             | Format Content           |                                                                                                                         |
| Format Content          | Code (JavaScript)     | Formats content into Markdown     | Extract Content         | Save to Google Drive     |                                                                                                                         |
| Save to Google Drive    | Google Drive          | Saves formatted content as file  | Format Content          | Wait Between Pages       |                                                                                                                         |
| Wait Between Pages      | Wait                  | Delays to respect server limits  | Save to Google Drive    | Loop URLs                |                                                                                                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node:**
   - Name: `Start Working Scraper`
   - Purpose: To start the workflow manually.

2. **Add a Set node:**
   - Name: `Set Sitemap URL`
   - Configure to set a string variable `sitemap_url` with the value of the target sitemap XML URL, e.g., `"https://yourwebsitehere.com/page-sitemap.xml"`.
   - Connect `Start Working Scraper` output to this node.

3. **Add an HTTP Request node:**
   - Name: `Get Sitemap`
   - Set method to GET.
   - URL: Set to expression `{{$json.sitemap_url}}`.
   - Timeout: 15000 ms.
   - In the response options, enable "Never Error" on failure responses to allow graceful handling.
   - Connect `Set Sitemap URL` output to this node.

4. **Add an XML node:**
   - Name: `Parse Sitemap XML`
   - Use default settings.
   - Connect `Get Sitemap` output to this node.

5. **Add a Split Out node:**
   - Name: `Split Sitemap URLs`
   - Set the field to split: `urlset.url`.
   - Connect `Parse Sitemap XML` output to this node.

6. **(Optional) Add a Limit node:**
   - Name: `Limit URLs (Optional)`
   - Set max items to 20 or desired limit.
   - Disable this node by default.
   - Connect `Split Sitemap URLs` output to this node.

7. **Add a Split In Batches node:**
   - Name: `Loop URLs`
   - Default batch size (1).
   - Connect the output of `Limit URLs (Optional)` (or directly from `Split Sitemap URLs` if limit disabled) to this node.

8. **Add an HTTP Request node:**
   - Name: `Scrape Page`
   - Method: GET.
   - URL: Expression: `{{$json.loc}}`
   - Timeout: 20000 ms.
   - Set HTTP headers to mimic a modern browser (User-Agent, Accept, Accept-Language, Accept-Encoding, Cache-Control, Pragma).
   - Enable "Never Error" on failure.
   - Connect `Loop URLs` output to this node.

9. **Add a Code node:**
   - Name: `Extract Content`
   - Language: JavaScript.
   - Insert the JavaScript code that:
     - Safely extracts the HTML content.
     - Extracts the page title using multiple fallback methods.
     - Cleans HTML to readable text.
     - Detects framework indicators (Divi, WordPress) and JavaScript presence.
     - Returns structured data including success flag and metadata.
   - Connect `Scrape Page` output to this node.

10. **Add another Code node:**
    - Name: `Format Content`
    - Language: JavaScript.
    - Insert JavaScript code that:
      - Formats input JSON into Markdown with YAML frontmatter.
      - Includes metadata, warnings for failures or limited content.
      - Structures content with headings and technical details.
    - Connect `Extract Content` output to this node.

11. **Add a Google Drive node:**
    - Name: `Save to Google Drive`
    - Operation: Create from text.
    - Name: Expression to generate file name by replacing `https://` or `http://` with blank and replacing `/` with `_` from current URL plus `_sitemap.md` suffix.
    - Content: Use `formatted` field from previous node.
    - Drive: Select "My Drive" or your Google Drive.
    - Folder ID: Set to your target folder ID (must be replaced by user).
    - Credentials: Configure Google Drive OAuth2 credentials.
    - Connect `Format Content` output to this node.

12. **Add a Wait node:**
    - Name: `Wait Between Pages`
    - Set wait time to 3 seconds.
    - Connect `Save to Google Drive` output to this node.

13. **Connect the Wait node back to `Loop URLs` to continue processing next batch.**

14. **Add a Sticky Note (optional) at the start:**
    - Name: `Working Scraper Note`
    - Content: Document the workflow’s features and purpose as described in section 1.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                | Context or Link                                                                                                  |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| The workflow respects scraping etiquette by implementing a 3-second delay between page requests to avoid server overload and reduce the risk of being blocked.              | General best practice for scraping workflows                                                                   |
| User must replace placeholders such as `https://yourwebsitehere.com/page-sitemap.xml`, `YOUR_GOOGLE_DRIVE_FOLDER_ID_HERE`, and Google Drive credentials with actual values. | Critical configuration before running the workflow                                                             |
| The scraper identifies if the site uses Divi or WordPress and flags possible JavaScript dependency issues to help diagnose scraping results.                               | Useful for troubleshooting content extraction limitations                                                      |
| Google Drive OAuth2 credentials require prior setup in n8n credentials manager with appropriate API scopes (read/write to Drive).                                         | n8n credential configuration guide: https://docs.n8n.io/credentials/google-drive/                                |
| The workflow is designed for static HTML content extraction; pages relying heavily on client-side JavaScript rendering may not be fully captured with this method.         | Consider integrating headless browser or Puppeteer nodes for JavaScript-rendered content if needed             |

---

**Disclaimer:**  
The provided content is extracted exclusively from an automated workflow created with n8n, a workflow automation tool. This process strictly respects active content policies and does not contain any illegal, offensive, or protected materials. All data handled is legal and public.