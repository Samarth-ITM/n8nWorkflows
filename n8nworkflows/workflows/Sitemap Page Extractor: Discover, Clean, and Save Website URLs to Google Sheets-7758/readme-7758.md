Sitemap Page Extractor: Discover, Clean, and Save Website URLs to Google Sheets

https://n8nworkflows.xyz/workflows/sitemap-page-extractor--discover--clean--and-save-website-urls-to-google-sheets-7758


# Sitemap Page Extractor: Discover, Clean, and Save Website URLs to Google Sheets

---

### 1. Workflow Overview

This workflow, titled **"Sitemap Page Extractor: Discover, Clean, and Save Website URLs to Google Sheets,"** is designed to automate the extraction of website page URLs by discovering sitemap files, parsing their content, filtering out irrelevant URLs, and saving the clean list to a Google Sheet for further use such as SEO audits or content analysis.

The workflow logically divides into the following blocks:

- **1.1 Input Reception and URL Preparation:** Receives user input (website URL) and formats it for sitemap discovery.
- **1.2 Sitemap URL Generation:** Constructs common sitemap URLs based on the domain.
- **1.3 Sitemap Validation and Discovery:** Checks which sitemap URLs exist and contain valid data.
- **1.4 Sitemap Index Processing:** Extracts nested sitemap URLs from sitemap index files or robots.txt.
- **1.5 Sitemap Page URL Extraction:** Retrieves page URLs listed inside each sitemap XML.
- **1.6 URL Filtering:** Removes unwanted URLs that include the term "sitemap" to avoid redundant or irrelevant links.
- **1.7 Data Storage:** Appends the cleaned page URLs into a Google Sheet, avoiding duplicates.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Input Reception and URL Preparation

**Overview:**  
Starts the workflow by accepting a website URL from a user-submitted form and prepares a standardized base URL for subsequent sitemap discovery.

**Nodes Involved:**  
- Input Website URL  
- Prepare website URL

**Node Details:**

- **Input Website URL**  
  - Type: Form Trigger  
  - Role: Entry point to receive the Website URL input from users.  
  - Configuration: Form titled "Sitemap Page Extractor" with a single field "Website URL." Uses a webhook trigger.  
  - Inputs: External user form submission.  
  - Outputs: JSON containing the submitted Website URL.  
  - Edge Cases: Missing or malformed URL input could cause errors downstream; no explicit validation here.  
  - Version: 2.2  

- **Prepare website URL**  
  - Type: Set node  
  - Role: Stores and formats the URL field from the form input into a consistent JSON property `url`.  
  - Configuration: Assigns `url = {{$json["Website URL"]}}`.  
  - Inputs: From "Input Website URL."  
  - Outputs: JSON with property `url`.  
  - Edge Cases: Does not sanitize or validate URL format; relies on user input correctness.  
  - Version: 3.4  

---

#### 2.2 Sitemap URL Generation

**Overview:**  
Generates a list of common sitemap URLs based on the input domain to attempt discovery of sitemap files.

**Nodes Involved:**  
- Build sitemap URLs  
- Sitemap URL Check (Split in Batches)

**Node Details:**

- **Build sitemap URLs**  
  - Type: Code node (JavaScript)  
  - Role: Constructs possible sitemap URLs like `/sitemap.xml`, `/sitemap_index.xml`, `/robots.txt`, and variations based on the domain extracted from the input URL.  
  - Configuration:  
    - Extracts domain (with protocol) from input URL, removes trailing slash.  
    - Creates an array of URLs including standard sitemap locations and robots.txt for additional sitemap discovery.  
  - Inputs: From "Prepare website URL."  
  - Outputs: Array of objects each with property `sitemap_url` for each constructed URL.  
  - Key Expression: JavaScript snippet carefully parses and standardizes the base domain, ensuring protocol presence.  
  - Edge Cases: Throws error if no URL provided; partial URLs or malformed input might yield incorrect domain extraction.  
  - Version: 2  

- **Sitemap URL Check**  
  - Type: SplitInBatches  
  - Role: Splits the array of sitemap URLs into batches for sequential processing to avoid rate limits or large concurrent requests.  
  - Inputs: From "Build sitemap URLs."  
  - Outputs: Passes individual sitemap URLs for HTTP requests.  
  - Edge Cases: Batch size defaults to n8n settings; large input arrays might need tuning.  
  - Version: 3  

---

#### 2.3 Sitemap Validation and Discovery

**Overview:**  
For each candidate sitemap URL, sends an HTTP request to verify existence and content presence, filtering out invalid or empty responses.

**Nodes Involved:**  
- Fetch Sitemap Data (HTTP Request)  
- Filter Non-Empty Sitemap Responses (If node)  
- Extract Sitemap URLs (Code node)

**Node Details:**

- **Fetch Sitemap Data**  
  - Type: HTTP Request  
  - Role: Sends GET request to each sitemap URL to fetch raw content (usually XML or robots.txt text).  
  - Configuration:  
    - URL dynamically set as `{{$json.sitemap_url}}`.  
    - Response format set to text.  
    - On error: Continue regular output to avoid workflow stop on 404 or other HTTP errors.  
  - Inputs: From "Sitemap URL Check."  
  - Outputs: Raw response data in property `data`.  
  - Edge Cases: HTTP failures, timeouts, or non-XML content may occur; handled by continuing output on error.  
  - Version: 4.2  

- **Filter Non-Empty Sitemap Responses**  
  - Type: If node  
  - Role: Filters responses to allow only those with a non-empty `data` property, ensuring only valid sitemap responses proceed.  
  - Configuration: Checks if `{{$json.data}}` is not empty string.  
  - Inputs: From "Fetch Sitemap Data."  
  - Outputs: Passes valid sitemap content forward.  
  - Edge Cases: Empty responses may result from 404 or empty files; these are excluded.  
  - Version: 2.2  

- **Extract Sitemap URLs**  
  - Type: Code node (JavaScript)  
  - Role: Parses sitemap index XML or robots.txt content to extract all nested sitemap URLs for further crawling.  
  - Configuration:  
    - Uses regex to find `Sitemap:` entries (robots.txt style) and `<loc>` tags (XML style).  
    - Aggregates all found URLs, removes duplicates.  
  - Inputs: From "Filter Non-Empty Sitemap Responses."  
  - Outputs: Array of objects each containing `sitemap_url` extracted from the content.  
  - Edge Cases: Malformed XML or unusual sitemap formats may miss some URLs; relies on regex matching.  
  - Version: 2  

---

#### 2.4 Sitemap Page URL Extraction

**Overview:**  
Fetches each discovered sitemap XML and extracts all listed page URLs for further processing.

**Nodes Involved:**  
- Fetch Sitemap Pages XML (HTTP Request)  
- Extract Page URLs from Sitemap (Code node)  

**Node Details:**

- **Fetch Sitemap Pages XML**  
  - Type: HTTP Request  
  - Role: Retrieves raw XML content of each sitemap URL containing actual page URLs.  
  - Configuration:  
    - URL dynamically set as `{{$json.sitemap_url}}`.  
    - GET method, response format text.  
    - On error: Continues output to prevent workflow failure on HTTP errors.  
  - Inputs: From "Extract Sitemap URLs."  
  - Outputs: Raw XML content in property `data`.  
  - Edge Cases: HTTP errors or invalid XML lead to empty or malformed data but do not stop workflow.  
  - Version: 4.2  

- **Extract Page URLs from Sitemap**  
  - Type: Code node (JavaScript)  
  - Role: Parses sitemap XML content to extract all individual page URLs within `<loc>` tags and anchor tags from any embedded HTML.  
  - Configuration:  
    - Uses regex to extract `<loc>` elements and `<a href="">` links.  
    - Aggregates unique URLs in a Set to avoid duplicates.  
    - Returns JSON objects each with `page_url`.  
  - Inputs: From "Fetch Sitemap Pages XML."  
  - Outputs: List of page URLs to be filtered.  
  - Edge Cases: If no URLs found, returns a message object indicating this; handles both XML and HTML content formats.  
  - Version: 2  

---

#### 2.5 URL Filtering

**Overview:**  
Filters out URLs that contain the word "sitemap" to exclude sitemap files themselves from the final page URL list.

**Nodes Involved:**  
- Exclude the Sitemap URLs (Filter node)

**Node Details:**

- **Exclude the Sitemap URLs**  
  - Type: Filter node  
  - Role: Applies a condition to exclude any URL containing the substring "sitemap" in the `page_url` property.  
  - Configuration: Filter condition uses string operation "not contains" with value "sitemap".  
  - Inputs: From "Extract Page URLs from Sitemap."  
  - Outputs: Passes only URLs without "sitemap" in their path.  
  - Edge Cases: Case-sensitive filtering; URLs with uppercase "SITEMAP" will pass through—may need enhancement for case insensitivity.  
  - Version: 2.2  

---

#### 2.6 Data Storage

**Overview:**  
Appends the filtered page URLs into a Google Sheet named "List_Of_All_URLs," updating existing entries to avoid duplicates.

**Nodes Involved:**  
- Save Page URLs to Sheet (Google Sheets node)

**Node Details:**

- **Save Page URLs to Sheet**  
  - Type: Google Sheets node  
  - Role: Appends or updates rows in a specified Google Sheet document with the extracted page URLs.  
  - Configuration:  
    - Operation: appendOrUpdate  
    - Sheet name: "List_Of_All_URLs"  
    - Matching column: "List URLs" to prevent duplicates  
    - Document ID: Set to the target Google Sheet URL (placeholder in JSON is "YOUR_GOOGLE_SHEET_URL")  
  - Inputs: From "Exclude the Sitemap URLs."  
  - Outputs: None (final output)  
  - Credentials: Uses OAuth2 credentials named "Shiv@incrementors.com - Google Sheets."  
  - Edge Cases: Google API quota limits, permission errors if credentials expire or lack access.  
  - Version: 4.6  

---

### 3. Summary Table

| Node Name                 | Node Type               | Functional Role                                                | Input Node(s)             | Output Node(s)                | Sticky Note                                                                                                                        |
|---------------------------|-------------------------|----------------------------------------------------------------|---------------------------|------------------------------|----------------------------------------------------------------------------------------------------------------------------------|
| Input Website URL          | Form Trigger            | Entry point to receive website URL from user                  | -                         | Prepare website URL           | ## Form Input & URL Preparation 📥 Workflow starts when user submits Website URL through form.                                    |
| Prepare website URL        | Set                     | Stores and formats the URL for sitemap discovery              | Input Website URL          | Build sitemap URLs            | ## Form Input & URL Preparation 🌐 The submitted URL is stored and formatted for processing.                                      |
| Build sitemap URLs         | Code                    | Generates common sitemap URLs based on domain                  | Prepare website URL        | Sitemap URL Check             | ## Build sitemap urls: Creates possible sitemap URLs like /sitemap.xml, /sitemap_index.xml, etc., for the domain.                |
| Sitemap URL Check          | SplitInBatches          | Splits sitemap URLs into batches for sequential processing     | Build sitemap URLs         | Filter Non-Empty Sitemap Responses, Fetch Sitemap Data | Sends HTTP requests to sitemap URLs and fetches responses to check sitemap existence and validity.                                |
| Fetch Sitemap Data         | HTTP Request            | Requests sitemap content for validation                        | Sitemap URL Check          | Filter Non-Empty Sitemap Responses | Sends an HTTP request to each sitemap URL and fetches raw response to verify sitemap existence and content.                      |
| Filter Non-Empty Sitemap Responses | If               | Filters out empty or invalid sitemap responses                 | Fetch Sitemap Data         | Extract Sitemap URLs          | ## Filter & Extract Sitemap Files Filters out unreachable or empty sitemap responses.                                             |
| Extract Sitemap URLs       | Code                    | Extracts nested sitemap URLs from robots.txt or sitemap index  | Filter Non-Empty Sitemap Responses | Fetch Sitemap Pages XML       | ## Filter & Extract Sitemap Files Parses sitemap index XML or robots.txt for nested sitemap links.                                 |
| Fetch Sitemap Pages XML    | HTTP Request            | Fetches XML content of each sitemap for page URLs extraction  | Extract Sitemap URLs       | Extract Page URLs from Sitemap | ## Fetching Sitemap XML and Extracting Page URLs Fetches sitemap XML content to extract listed pages.                             |
| Extract Page URLs from Sitemap | Code                 | Parses sitemap XML to extract page URLs                        | Fetch Sitemap Pages XML    | Exclude the Sitemap URLs      | ## Fetching Sitemap XML and Extracting Page URLs Parses XML data to get individual page URLs.                                      |
| Exclude the Sitemap URLs   | Filter                  | Removes URLs containing "sitemap" to exclude sitemap files    | Extract Page URLs from Sitemap | Save Page URLs to Sheet       | ## Exclude Sitemap URLs Filters out URLs containing the word "sitemap".                                                           |
| Save Page URLs to Sheet    | Google Sheets           | Appends or updates extracted page URLs in Google Sheets       | Exclude the Sitemap URLs   | -                            | Appends each crawled page URL into Google Sheets "List_Of_All_URLs" sheet, avoiding duplicates.                                    |
| Sticky Note                | Sticky Note             | Documentation block                                            | -                         | -                            | ## Build sitemap urls: Creates possible sitemap URLs like /sitemap.xml, /sitemap_index.xml, etc., for the domain.                 |
| Sticky Note1               | Sticky Note             | Documentation block                                            | -                         | -                            | ## Filter & Extract Sitemap Files: Filters out empty sitemap responses and extracts nested sitemap URLs.                          |
| Sticky Note2               | Sticky Note             | Documentation block                                            | -                         | -                            | ## Fetching Sitemap XML and Extracting Page URLs: Fetches sitemap XML and extracts page URLs inside.                               |
| Sticky Note3               | Sticky Note             | Documentation block                                            | -                         | -                            | Append each crawled page URL into the List_Of_All_URLs sheet, avoiding duplicates by matching existing entries automatically.    |
| Sticky Note4               | Sticky Note             | Documentation block                                            | -                         | -                            | ## Form Input & URL Preparation: Workflow starts when user submits Website URL and formats it.                                     |
| Sticky Note5               | Sticky Note             | Documentation block                                            | -                         | -                            | Sends an HTTP request to each generated sitemap URL and fetches raw response to check if sitemap exists and contains valid data.  |
| Sticky Note6               | Sticky Note             | Documentation block                                            | -                         | -                            | ## Exclude Sitemap URLs: Filters out URLs containing the word "sitemap" to remove unwanted entries.                               |
| Sticky Note7               | Sticky Note             | Documentation block                                            | -                         | -                            | ## 📌 Automation Summary: Detailed workflow purpose, flow, and support contact info.                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "Input Website URL" node:**  
   - Type: Form Trigger  
   - Configure form with title "Sitemap Page Extractor" and one field labeled "Website URL."  
   - Use webhook trigger to start workflow on form submission.

2. **Create "Prepare website URL" node:**  
   - Type: Set  
   - Assign field `url` with expression: `{{$json["Website URL"]}}`  
   - Connect from "Input Website URL."

3. **Create "Build sitemap URLs" node:**  
   - Type: Code (JavaScript)  
   - Paste the following JS code to generate sitemap URLs:

```javascript
const inputData = $input.first().json;
let baseUrl = inputData.url || inputData.website_url || '';

if (!baseUrl) {
  throw new Error("No URL provided");
}

baseUrl = baseUrl.replace(/\/$/, '');

let domain = '';
if (baseUrl.includes('://')) {
  const parts = baseUrl.split('/');
  domain = parts[0] + '//' + parts[2];
} else {
  domain = 'https://' + baseUrl;
}

const urls = [
  `${domain}/robots.txt`,
  `${domain}/sitemap.xml`,
  `${domain}/sitemap_index.xml`,
  `${domain}/sitemap-index.xml`,
  `${domain}/sitemap1.xml`,
  `${domain}/sitemap/sitemap.xml`,
  `${domain}/sitemaps/sitemap.xml`
];

return urls.map(url => ({ sitemap_url: url }));
```

   - Connect from "Prepare website URL."

4. **Create "Sitemap URL Check" node:**  
   - Type: SplitInBatches  
   - Default settings suffice.  
   - Connect from "Build sitemap URLs."

5. **Create "Fetch Sitemap Data" node:**  
   - Type: HTTP Request  
   - URL: `{{$json.sitemap_url}}`  
   - Method: GET  
   - Response Format: Text  
   - On Error: Continue Regular Output  
   - Connect from "Sitemap URL Check."

6. **Create "Filter Non-Empty Sitemap Responses" node:**  
   - Type: If  
   - Condition: Check if `{{$json.data}}` is not empty string (operator: string notEmpty).  
   - Connect from "Fetch Sitemap Data."

7. **Create "Extract Sitemap URLs" node:**  
   - Type: Code (JavaScript)  
   - Paste this code:

```javascript
const allUrls = [];

$input.all().forEach(item => {
  const content = item.json.data || '';

  const robotMatches = [...content.matchAll(/Sitemap:\s*(\S+)/gi)];
  robotMatches.forEach(match => {
    allUrls.push(match[1]);
  });

  const locMatches = [...content.matchAll(/<loc>\s*(.*?)\s*<\/loc>/gi)];
  locMatches.forEach(match => {
    allUrls.push(match[1]);
  });
});

const uniqueUrls = [...new Set(allUrls)];

return uniqueUrls.map(url => ({ json: { sitemap_url: url }}));
```

   - Connect from "Filter Non-Empty Sitemap Responses" (true branch).

8. **Create "Fetch Sitemap Pages XML" node:**  
   - Type: HTTP Request  
   - URL: `{{$json.sitemap_url}}`  
   - Method: GET  
   - Response Format: Text  
   - On Error: Continue Regular Output  
   - Connect from "Extract Sitemap URLs."

9. **Create "Extract Page URLs from Sitemap" node:**  
   - Type: Code (JavaScript)  
   - Paste:

```javascript
const urls = new Set();

$input.all().forEach(item => {
  const content = item.json.data || '';

  const xmlMatches = [...content.matchAll(/<loc>(.*?)<\/loc>/gi)];
  xmlMatches.forEach(match => {
    const url = match[1].trim();
    urls.add(url);
  });

  const htmlMatches = [...content.matchAll(/<a\s[^>]*href=["']([^"']+)["']/gi)];
  htmlMatches.forEach(match => {
    const url = match[1].trim();
    if (url.startsWith('/') || url.startsWith('http')) {
      urls.add(url);
    }
  });
});

if (urls.size === 0) {
  return [{ json: { message: "No URLs found in XML or HTML content" }}];
}

return Array.from(urls).map(url => ({
  json: { page_url: url }
}));
```

   - Connect from "Fetch Sitemap Pages XML."

10. **Create "Exclude the Sitemap URLs" node:**  
    - Type: Filter  
    - Condition: `page_url` does NOT contain "sitemap" (string operator: not contains, case-sensitive).  
    - Connect from "Extract Page URLs from Sitemap."

11. **Create "Save Page URLs to Sheet" node:**  
    - Type: Google Sheets  
    - Operation: appendOrUpdate  
    - Sheet Name: "List_Of_All_URLs"  
    - Document ID: Enter your Google Sheet URL  
    - Matching Columns: "List URLs" (to avoid duplicates)  
    - Credentials: Configure OAuth2 credentials with access to the Google Sheet.  
    - Connect from "Exclude the Sitemap URLs."

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                  | Context or Link                                  |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------|
| This automation simplifies the collection of website page URLs from sitemap URLs, enabling SEO audits, content analysis, redirects, or reporting.                                                                                                                                                           | Sticky Note7 content summary                      |
| For support or questions regarding this workflow, contact info@incrementors.com or fill out the form at https://www.incrementors.com/contact-us/                                                                                                                                                            | Support contact info                              |
| The workflow uses common sitemap URL patterns and also checks robots.txt files for sitemap declarations to maximize discovery coverage.                                                                                                                                                                     | Implied in sitemap URL generation and extraction |
| The Google Sheets node requires valid OAuth2 credentials with permission to edit the target sheet. Ensure the Google Sheet has a sheet named "List_Of_All_URLs" with a column labeled "List URLs" for matching duplicates.                                                                                      | Google Sheets node configuration                  |
| URL filtering is case-sensitive and filters out URLs containing the string "sitemap" exactly; consider updating to case-insensitive if needed.                                                                                                                                                               | Exclude the Sitemap URLs node                     |
| HTTP Request nodes have error handling set to continue on error to avoid workflow interruption when sitemap URLs do not exist or are inaccessible.                                                                                                                                                            | HTTP Request nodes (Fetch Sitemap Data, Fetch Sitemap Pages XML) |
| The workflow does not explicitly validate user input URLs; input sanitization or validation should be added externally if needed.                                                                                                                                                                            | Input Website URL and Prepare website URL nodes  |

---

**Disclaimer:**  
The provided text is extracted exclusively from an automated n8n workflow. It complies strictly with current content policies and contains no illegal, offensive, or protected content. All data processed is legal and public.

---