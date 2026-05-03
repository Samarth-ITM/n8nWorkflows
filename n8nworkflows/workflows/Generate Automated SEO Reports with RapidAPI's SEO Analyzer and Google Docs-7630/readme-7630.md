Generate Automated SEO Reports with RapidAPI's SEO Analyzer and Google Docs

https://n8nworkflows.xyz/workflows/generate-automated-seo-reports-with-rapidapi-s-seo-analyzer-and-google-docs-7630


# Generate Automated SEO Reports with RapidAPI's SEO Analyzer and Google Docs

### 1. Workflow Overview

This workflow automates the generation of SEO audit reports for websites submitted via a form. It targets digital marketers, SEO specialists, and webmasters who need quick, automated SEO assessments of any website URL. The workflow includes the following logical blocks:

- **1.1 Input Reception:** Captures a website URL from user input via a form.
- **1.2 SEO Data Retrieval:** Sends the URL to RapidAPI’s SEO Analyzer API to obtain comprehensive SEO audit data.
- **1.3 Data Processing and Formatting:** Parses and reformats the raw API response into a structured, readable Markdown report summarizing key SEO metrics.
- **1.4 Report Delivery:** Inserts the formatted SEO report into a designated Google Docs document for easy access and sharing.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
This block listens for form submissions containing a website URL and triggers the workflow with this input.

**Nodes Involved:**  
- On form submission

**Node Details:**

- **Node Name:** On form submission  
- **Type:** `formTrigger` (Webhook/Form Trigger)  
- **Configuration:**  
  - Form titled "Website Audit" with a single required field labeled "url".  
  - No additional options enabled.  
- **Key Expressions/Variables:**  
  - Extracts `url` from the form submission JSON payload (`$json.url`).  
- **Input Connections:** None (start node).  
- **Output Connections:** Connects to “Website Audit”.  
- **Version Requirements:** Version 2.2 or later recommended for formTrigger stability.  
- **Potential Failures:**  
  - Missing or invalid URL submission (handled downstream).  
  - Webhook connectivity issues or delays.  
- **Sub-workflow:** None.

---

#### 2.2 SEO Data Retrieval

**Overview:**  
Sends a POST request with the submitted URL to the SEO Analyzer API on RapidAPI to retrieve detailed SEO audit data.

**Nodes Involved:**  
- Website Audit

**Node Details:**

- **Node Name:** Website Audit  
- **Type:** `httpRequest`  
- **Configuration:**  
  - HTTP Method: POST  
  - URL: `https://website-seo-analyzer-and-audit-ai.p.rapidapi.com/seo.php`  
  - Content Type: `multipart/form-data`  
  - Body includes a parameter named `url` with the value taken from the form submission (`={{ $json.url }}`).  
  - Headers include `x-rapidapi-host` and `x-rapidapi-key` (the key must be replaced with a valid RapidAPI key).  
- **Key Expressions/Variables:**  
  - Dynamic URL value from input: `={{ $json.url }}`  
- **Input Connections:** From “On form submission”.  
- **Output Connections:** To “Reformat”.  
- **Version Requirements:** Version 4.2 or higher recommended for multipart form data support.  
- **Potential Failures:**  
  - API authentication errors due to invalid or missing API key.  
  - Network timeouts or endpoint unavailability.  
  - Invalid URL format causing API errors.  
  - Rate limiting by RapidAPI.  
- **Sub-workflow:** None.

---

#### 2.3 Data Processing and Formatting

**Overview:**  
Processes the raw SEO audit JSON data to extract relevant metrics and formats them into a clean Markdown report summarizing metadata, keyword density, headers, image optimization, links, performance, security, and structured data.

**Nodes Involved:**  
- Reformat

**Node Details:**

- **Node Name:** Reformat  
- **Type:** `code` (JavaScript)  
- **Configuration:**  
  - Uses JavaScript to parse API response nested under `data.apiData.results`.  
  - Extracts sections such as basic metadata, advanced metrics, performance, and security.  
  - Formats boolean statuses into checkmarks (✅) or crosses (❌) for readability.  
  - Aggregates keyword stats, header tags, image alt tag counts, and links.  
  - Builds a Markdown string with headers and bullet points for each SEO aspect.  
- **Key Expressions/Variables:**  
  - Accesses `basic`, `adv`, `perf`, and `sec` from the API response.  
  - Uses helper function `formatStatus` to convert statuses.  
  - Constructs variables like `keywordStats`, `titleKeywords`, `descKeywords`.  
- **Input Connections:** From “Website Audit”.  
- **Output Connections:** To “Add Data In Google Docs”.  
- **Version Requirements:** Version 2 for `code` node supporting modern JS syntax.  
- **Potential Failures:**  
  - Null or undefined fields if API response structure changes.  
  - Errors when accessing nested properties if data is missing.  
  - Syntax errors in JavaScript code.  
- **Sub-workflow:** None.

---

#### 2.4 Report Delivery

**Overview:**  
Inserts the formatted SEO report Markdown into a specified Google Docs document using Google Docs API via a service account.

**Nodes Involved:**  
- Add Data In Google Docs

**Node Details:**

- **Node Name:** Add Data In Google Docs  
- **Type:** `googleDocs`  
- **Configuration:**  
  - Operation: `update`  
  - Action: Insert text specified by `docContent` from previous node.  
  - Document URL field is empty and must be set to target Google Docs file URL.  
  - Authenticated via Service Account credentials named "Google Docs account".  
- **Key Expressions/Variables:**  
  - Inserts the Markdown report from `$json.docContent`.  
- **Input Connections:** From “Reformat”.  
- **Output Connections:** None (end node).  
- **Version Requirements:** Version 2 or higher for Google Docs node.  
- **Potential Failures:**  
  - Authentication errors if service account credentials are misconfigured.  
  - Empty or incorrect Google Docs URL causing update failures.  
  - API rate limits or quota exceeded errors.  
- **Sub-workflow:** None.

---

### 3. Summary Table

| Node Name           | Node Type       | Functional Role                               | Input Node(s)          | Output Node(s)        | Sticky Note                                                                                      |
|---------------------|-----------------|-----------------------------------------------|-----------------------|-----------------------|-------------------------------------------------------------------------------------------------|
| On form submission  | formTrigger     | Captures website URL input via form            | None                  | Website Audit          | 🟢 **On form submission** - Captures the website URL entered by the user in the form to initiate the audit process.           |
| Website Audit       | httpRequest     | Sends URL to SEO Analyzer API via RapidAPI    | On form submission    | Reformat               | 🌐 **Website Audit** - Sends a POST request with the submitted URL to the SEO Analyzer API via RapidAPI to retrieve audit data. |
| Reformat            | code            | Formats raw SEO JSON data into Markdown report| Website Audit         | Add Data In Google Docs| 🧠 **Reformat** - Extracts and formats the raw SEO data into structured, readable Markdown report summarizing key metrics.       |
| Add Data In Google Docs | googleDocs  | Inserts the formatted report into Google Docs | Reformat              | None                   | 📄 **Add Data In Google Docs** - Appends the formatted SEO report into a specified Google Docs document using connected account.  |
| Sticky Note         | stickyNote      | Documentation and explanation                   | None                  | None                   | ### 🧾 Automated Website SEO Audit and Google Docs Report... (Full content in node)                |
| Sticky Note1        | stickyNote      | Description of On form submission node          | None                  | None                   | 🟢 **On form submission** - Captures the website URL entered by the user in the form to initiate the audit process.              |
| Sticky Note2        | stickyNote      | Description of Website Audit node                | None                  | None                   | 🌐 **Website Audit** - Sends a POST request with the submitted URL to the SEO Analyzer API via RapidAPI to retrieve audit data.  |
| Sticky Note3        | stickyNote      | Description of Reformat node                      | None                  | None                   | 🧠 **Reformat** - Extracts and formats the raw SEO data into a structured, readable Markdown report summarizing performance, metadata, security, and more. |
| Sticky Note5        | stickyNote      | Description of Add Data In Google Docs node      | None                  | None                   | 📄 **Add Data In Google Docs** - Appends the formatted SEO report into a specified Google Docs document using the connected Google account. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the “On form submission” node:**  
   - Type: `formTrigger`  
   - Configure form title: "Website Audit"  
   - Add one required field: `url` (type: text)  
   - Save and activate webhook to receive submissions.  

2. **Create the “Website Audit” node:**  
   - Type: `httpRequest`  
   - Set HTTP Method: POST  
   - URL: `https://website-seo-analyzer-and-audit-ai.p.rapidapi.com/seo.php`  
   - Set Content Type: `multipart/form-data`  
   - Add body parameter: `url` with expression `={{ $json.url }}`  
   - Add header parameters:  
     - `x-rapidapi-host` = `website-seo-analyzer-and-audit-ai.p.rapidapi.com`  
     - `x-rapidapi-key` = *your RapidAPI key here*  
   - Connect input from “On form submission”.  

3. **Create the “Reformat” node:**  
   - Type: `code` (JavaScript) node  
   - Paste the provided JavaScript code to parse and format the SEO data into Markdown.  
   - Connect input from “Website Audit”.  

4. **Create the “Add Data In Google Docs” node:**  
   - Type: `googleDocs`  
   - Operation: `update`  
   - Action: Insert text with expression: `={{ $json.docContent }}`  
   - Provide the URL of the target Google Docs document in the “Document URL” field.  
   - Set authentication method to Service Account and select or create Google API credentials with Docs API enabled.  
   - Connect input from “Reformat”.  

5. **Link nodes in this order:**  
   `On form submission` → `Website Audit` → `Reformat` → `Add Data In Google Docs`.  

6. **Credential setup:**  
   - Create RapidAPI credential with your API key for the SEO Analyzer service.  
   - Configure Google API credentials as a Service Account with appropriate scopes for Google Docs API.  

7. **Testing:**  
   - Submit a test URL through the form.  
   - Verify that the SEO audit report populates correctly in the specified Google Doc.  
   - Handle errors such as invalid URLs, API key issues, or permission denials.  

---

### 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                                                                                          |
|---------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| This workflow automates website SEO audits and integrates results directly into Google Docs for easy sharing. | Main project goal and description.                                                                      |
| To use the workflow, obtain a RapidAPI key for the SEO Analyzer API: https://rapidapi.com/                     | RapidAPI service for SEO auditing.                                                                      |
| Google Docs API requires enabling in Google Cloud Console and Service Account with Docs API scopes.            | Credential setup instructions for Google Docs integration.                                              |
| The Markdown report includes detailed SEO metrics such as metadata, keyword density, headers, images, links.   | Report content structure and formatting guidelines.                                                     |
| The formTrigger node requires n8n version 0.151.0 or newer for stable form-based input handling.                | Version-specific note for formTrigger node.                                                             |

---

**Disclaimer:** The provided text is exclusively derived from an automated workflow created with n8n, a tool for integration and automation. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and publicly accessible.