Sitemap.xml Metadata Export to Google Sheets (Title, Meta Description, and URLs)

https://n8nworkflows.xyz/workflows/sitemap-xml-metadata-export-to-google-sheets--title--meta-description--and-urls--8691


# Sitemap.xml Metadata Export to Google Sheets (Title, Meta Description, and URLs)

### 1. Workflow Overview

This workflow automates the extraction of SEO-relevant metadata—specifically the `<title>`, meta description, and URLs—from every page listed in a website’s sitemap.xml and exports this data into a Google Sheets spreadsheet. It is designed primarily for SEO audits, content inventories, and website migration planning.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Sitemap Retrieval:** Trigger the workflow manually, fetch the sitemap XML from a user-configured URL.
- **1.2 URL Extraction and Iteration:** Parse the sitemap XML to extract URLs and iterate over each URL in batches.
- **1.3 HTML Retrieval and Metadata Extraction:** For each URL, request the page’s HTML, extract the page title and meta description.
- **1.4 Data Aggregation and Google Sheets Update:** Merge extracted metadata with URLs and append or update rows in the configured Google Sheets document.
- **1.5 Rate Limiting Pause:** Introduce a wait time between requests to avoid server rate limiting.

### 2. Block-by-Block Analysis

---

#### 2.1 Input Reception and Sitemap Retrieval

**Overview:**  
Starts the workflow manually and fetches the sitemap XML from a configurable URL.

**Nodes Involved:**  
- When clicking ‘Execute workflow’  
- Get Sitemap XML

**Node Details:**

- **When clicking ‘Execute workflow’**  
  - *Type:* Manual Trigger  
  - *Role:* Entry point for the workflow; user manually triggers execution.  
  - *Configuration:* No special parameters.  
  - *Connections:* Output feeds into “Get Sitemap XML.”  
  - *Edge Cases:* User must manually start; no automatic scheduling.  
  - *Version:* Standard, no version-specific requirements.

- **Get Sitemap XML**  
  - *Type:* HTTP Request  
  - *Role:* Retrieves the sitemap XML via HTTP GET.  
  - *Configuration:*  
    - URL is set by default to `https://example.com/sitemap.xml` and must be replaced with the target sitemap URL.  
    - No authentication or special headers.  
  - *Key Expressions:* None, static URL.  
  - *Connections:* Receives input from manual trigger; outputs raw XML data to “Split the title and url.”  
  - *Edge Cases:*  
    - 404 or network errors if URL is incorrect or unreachable.  
    - Sitemap format errors if XML is malformed.  
  - *Sticky Note:* Reminds user to replace the example sitemap URL with their actual sitemap URL.

---

#### 2.2 URL Extraction and Iteration

**Overview:**  
Extracts all URLs from the sitemap XML and iterates over them individually or in batches for subsequent processing.

**Nodes Involved:**  
- Split the title and url  
- Loop Over Items

**Node Details:**

- **Split the title and url**  
  - *Type:* Code (JavaScript)  
  - *Role:* Parses raw sitemap XML, extracts all URLs from `<loc>` tags into individual workflow items.  
  - *Configuration:*  
    - Uses a global regex `/\<loc\>(.*?)\<\/loc\>/g` to find all URL entries.  
    - Emits one output item per URL.  
  - *Input:* Raw XML string from “Get Sitemap XML.”  
  - *Output:* List of items each containing a single URL under `json.loc`.  
  - *Edge Cases:*  
    - Sitemap with no `<loc>` tags results in empty output.  
    - XML encoding issues could cause regex failure.  
  - *Version:* Uses n8n v2 Code node syntax.

- **Loop Over Items**  
  - *Type:* SplitInBatches  
  - *Role:* Processes URLs in batches (default batch size).  
  - *Configuration:* Default options; batch size not explicitly set (uses default).  
  - *Input:* List of URL items from “Split the title and url.”  
  - *Output:* Iterates URLs one by one to downstream nodes.  
  - *Edge Cases:* Large sitemaps may slow processing; no explicit batch size control here.  
  - *Connections:* Feeds into “Get specific link html” (main output) and merges with meta extraction downstream.

---

#### 2.3 HTML Retrieval and Metadata Extraction

**Overview:**  
Fetches the HTML content of each URL and extracts the HTML `<title>` and meta description content.

**Nodes Involved:**  
- Get specific link html  
- Get the meta description  
- Merge Meta Description with Title and URL

**Node Details:**

- **Get specific link html**  
  - *Type:* HTTP Request  
  - *Role:* Fetches HTML content of each URL from the batch.  
  - *Configuration:*  
    - URL dynamically set to `{{ $json.loc }}` — the URL extracted from sitemap.  
    - Standard GET request, no authentication.  
  - *Input:* Iterated URL from “Loop Over Items.”  
  - *Output:* Raw HTML content of the page.  
  - *Edge Cases:*  
    - 404 or 500 errors if page unavailable.  
    - Slow responses or timeouts may occur.  
    - Potential blocking by server if too frequent calls.  
  - *Sticky Note:* Advises rate limiting to avoid 429 errors.

- **Get the meta description**  
  - *Type:* Code (JavaScript)  
  - *Role:* Parses HTML string to extract `<title>` and `<meta name="description">` content.  
  - *Configuration:*  
    - Uses regex to find `<title>` content.  
    - Uses regex to find meta description content attribute.  
    - Returns a single JSON with `title` and `description` fields.  
  - *Input:* Raw HTML from “Get specific link html.”  
  - *Output:* Object containing `title` and `description` properties.  
  - *Edge Cases:*  
    - Missing `<title>` or meta description tags result in empty strings.  
    - Malformed HTML could break regex extraction.  
  - *Version:* Uses n8n v2 Code node syntax.

- **Merge Meta Description with Title and URL**  
  - *Type:* Merge  
  - *Role:* Combines the original URL data with extracted metadata into one item.  
  - *Configuration:*  
    - Mode: Combine by position (pairs items from both inputs based on index).  
  - *Input:*  
    - URL item from “Loop Over Items” (index 1)  
    - Metadata from “Get the meta description” (index 0)  
  - *Output:* Single combined item containing URL, title, and description.  
  - *Edge Cases:* Mismatched item counts could cause incorrect pairing.  
  - *Version:* Standard merge node.

---

#### 2.4 Data Aggregation and Google Sheets Update

**Overview:**  
Appends or updates each row in a Google Sheet with the URL, title, and meta description extracted.

**Nodes Involved:**  
- Append or update row in sheet  
- Wait

**Node Details:**

- **Append or update row in sheet**  
  - *Type:* Google Sheets  
  - *Role:* Adds new row or updates existing row based on URL match in Google Sheets.  
  - *Configuration:*  
    - Operation: appendOrUpdate  
    - Document ID and Sheet Name configured with a specific target spreadsheet and sheet tab.  
    - Columns mapped to:  
      - URL = `{{ $json.loc }}`  
      - Title = `{{ $json.title }}`  
      - meta description = `{{ $json.description }}`  
    - Matching column for update: URL (prevents duplicates).  
  - *Credentials:* Google Sheets OAuth2 credentials required.  
  - *Input:* Combined metadata object from the merge node.  
  - *Output:* Passes to the “Wait” node.  
  - *Edge Cases:*  
    - Authentication errors if credentials expire or are invalid.  
    - API quota limits or request failures.  
    - Spreadsheet schema mismatch (missing columns) causes errors.  
  - *Sticky Note:* Instructions to set up credentials, document ID, sheet name, and column schema.

- **Wait**  
  - *Type:* Wait  
  - *Role:* Adds a 2-second pause between each Google Sheets update to avoid rate limiting or server blocks.  
  - *Configuration:* 2 seconds delay.  
  - *Input:* From Google Sheets node.  
  - *Output:* Loops back to “Loop Over Items” to process next URL batch.  
  - *Edge Cases:* Too short delay may cause 429 errors; increase delay if needed.  
  - *Sticky Note:* Specifies this node’s importance for rate limiting.

---

### 3. Summary Table

| Node Name                      | Node Type                 | Functional Role                           | Input Node(s)                    | Output Node(s)                  | Sticky Note                                                                                                                      |
|-------------------------------|---------------------------|-----------------------------------------|---------------------------------|--------------------------------|----------------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger            | Workflow entry point                     | —                               | Get Sitemap XML                 |                                                                                                                                  |
| Get Sitemap XML                | HTTP Request              | Fetch sitemap.xml from configured URL   | When clicking ‘Execute workflow’ | Split the title and url         | ⚙️ Configuration Required: Replace example sitemap URL with your actual sitemap URL.                                            |
| Split the title and url        | Code                      | Extract URLs from sitemap XML            | Get Sitemap XML                 | Loop Over Items                 |                                                                                                                                  |
| Loop Over Items               | SplitInBatches            | Iterate over URLs in batches             | Split the title and url         | Get specific link html, Merge Meta Description with Title and URL |                                                                                                                                  |
| Get specific link html         | HTTP Request              | Fetch HTML content of each URL           | Loop Over Items                 | Get the meta description        | ⏸️ Rate Limiting: Use Wait node to pause between requests to avoid 429 Too Many Requests errors.                                  |
| Get the meta description       | Code                      | Extract <title> and meta description     | Get specific link html          | Merge Meta Description with Title and URL |                                                                                                                                  |
| Merge Meta Description with Title and URL | Merge                     | Combine URL and metadata into one item    | Loop Over Items, Get the meta description | Append or update row in sheet |                                                                                                                                  |
| Append or update row in sheet  | Google Sheets             | Append or update metadata in Google Sheet | Merge Meta Description with Title and URL | Wait                          | ⚙️ Configuration Required: Set up Google Sheets OAuth2 credentials, specify document ID, sheet name, and columns (URL, Title, meta description). |
| Wait                          | Wait                      | Pause between requests to avoid rate limits | Append or update row in sheet  | Loop Over Items                | ⏸️ Rate Limiting: Default 2 seconds delay; increase if 429 errors occur.                                                         |
| Sticky Note                   | Sticky Note               | Workflow overview and purpose explanation | —                               | —                              | # 🔍 Sitemap Metadata Exporter. Instructions on use and purpose.                                                                  |
| Sticky Note1                  | Sticky Note               | Configuration instructions                | —                               | —                              | # ⚙️ Configuration Required: Replace sitemap URL and configure Google Sheets credentials and columns.                            |
| Sticky Note3                  | Sticky Note               | Rate limiting instructions                | —                               | —                              | # ⏸️ Rate Limiting: Use Wait node to prevent server blocking and 429 errors.                                                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger Node**  
   - Name: `When clicking ‘Execute workflow’`  
   - Purpose: To manually start the workflow.

2. **Create an HTTP Request Node to Fetch Sitemap**  
   - Name: `Get Sitemap XML`  
   - Set Method: GET  
   - URL: Replace default `https://example.com/sitemap.xml` with your actual sitemap URL.  
   - No authentication or special headers required.  
   - Connect output of manual trigger node to this node.

3. **Create a Code Node to Extract URLs from Sitemap**  
   - Name: `Split the title and url`  
   - Language: JavaScript  
   - Code:  
     ```javascript
     const xml = items[0].json.body || items[0].json.data;
     const urls = Array.from(
       xml.matchAll(/<loc>(.*?)<\/loc>/g),
       match => match[1]
     );
     return urls.map(url => ({ json: { loc: url } }));
     ```  
   - Connect output of “Get Sitemap XML” node to this node.

4. **Create a SplitInBatches Node**  
   - Name: `Loop Over Items`  
   - Use default batch size or customize if desired.  
   - Connect output of code node to this node.

5. **Create an HTTP Request Node to Fetch Each Page's HTML**  
   - Name: `Get specific link html`  
   - Method: GET  
   - URL: Use expression `{{$json.loc}}` to dynamically fetch the current URL.  
   - Connect main output of SplitInBatches node to this node.

6. **Create a Code Node to Extract Title and Meta Description**  
   - Name: `Get the meta description`  
   - Language: JavaScript  
   - Code:  
     ```javascript
     const html = items[0].json.body || items[0].json.data;
     const titleMatch = html.match(/<title>([\s\S]*?)<\/title>/i);
     const title = titleMatch ? titleMatch[1].trim() : '';
     const descMatch = html.match(/<meta\b[^>]*\bname=["']description["'][^>]*\bcontent=["']([\s\S]*?)["']/i);
     const description = descMatch ? descMatch[1].trim() : '';
     return [{ json: { title, description } }];
     ```  
   - Connect output of “Get specific link html” to this node.

7. **Create a Merge Node**  
   - Name: `Merge Meta Description with Title and URL`  
   - Mode: Combine  
   - Combine By: Position  
   - Connect the second output (index 1) of the SplitInBatches node to input 2 of this merge node (to bring URL data).  
   - Connect output of “Get the meta description” node to input 1 of this merge node.

8. **Create a Google Sheets Node to Append or Update Rows**  
   - Name: `Append or update row in sheet`  
   - Operation: appendOrUpdate  
   - Document ID: Paste your Google Sheets document ID.  
   - Sheet Name: Select the target sheet/tab by name or ID.  
   - Columns Mapping:  
     - URL → `{{$json.loc}}`  
     - Title → `{{$json.title}}`  
     - meta description → `{{$json.description}}`  
   - Matching Columns: `URL` (to prevent duplicates)  
   - Credentials: Set up and select Google Sheets OAuth2 credentials.  
   - Connect output of Merge node to this node.

9. **Create a Wait Node**  
   - Name: `Wait`  
   - Duration: 2 seconds (adjustable if rate limiting occurs)  
   - Connect output of Google Sheets node to this node.

10. **Connect Wait Node Back to SplitInBatches Node**  
    - Connect output of Wait node back to input of “Loop Over Items” node to continue processing remaining URLs.

---

### 5. General Notes & Resources

| Note Content                                                                                                        | Context or Link                                                                                                                       |
|--------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------|
| This workflow automates extraction of page titles and meta descriptions from sitemap URLs for SEO and content audit. | Workflow purpose and usage instructions.                                                                                            |
| Replace the example sitemap URL (`https://example.com/sitemap.xml`) with your actual sitemap URL before running.    | Configuration requirement to avoid workflow failure.                                                                                 |
| Ensure Google Sheets OAuth2 credentials are configured in n8n for access to your target spreadsheet.                 | Credential setup for Google Sheets node.                                                                                            |
| Use the Wait node to pause 2 seconds between page requests to avoid HTTP 429 Too Many Requests errors.               | Rate limiting advice to prevent server blocks and API throttling.                                                                   |
| Sheet columns must be named exactly: `URL`, `Title`, and `meta description` for proper data mapping.                | Google Sheets schema requirement for the Append or update row node.                                                                 |

---

**Disclaimer:** The provided text is exclusively derived from an n8n automation workflow. The process complies strictly with content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.