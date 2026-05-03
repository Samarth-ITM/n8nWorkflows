Extract Website URLs from Sitemap.XML for SEO Analysis

https://n8nworkflows.xyz/workflows/extract-website-urls-from-sitemap-xml-for-seo-analysis-4671


# Extract Website URLs from Sitemap.XML for SEO Analysis

### 1. Workflow Overview

This workflow automates the extraction of website URLs from a sitemap.xml file for SEO analysis purposes. It is designed to handle both simple sitemaps and sitemap indexes containing multiple sub-sitemaps, enabling comprehensive URL extraction from large or segmented sitemaps.

The workflow consists of the following logical blocks:

- **1.1 Input Initialization:** Receives manual trigger input and sets the base website URL or accepts a direct sitemap URL.
- **1.2 Sitemap Download and Parsing:** Downloads the sitemap.xml or sitemap index XML, parses it, and splits sitemap indexes into individual sitemap URLs.
- **1.3 Sub-Sitemap Processing:** Downloads each sub-sitemap (if present), parses it, and extracts all URLs listed.
- **1.4 Output Generation:** Converts the extracted URLs into downloadable files or prepares data for alternative outputs such as Google Sheets.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Initialization

- **Overview:**  
  This block initializes the workflow by accepting a manual trigger, setting the base domain URL, or optionally allowing pasting a direct sitemap URL. It establishes the starting point for sitemap retrieval.

- **Nodes Involved:**  
  - When clicking ‘Test workflow’  
  - Set URL  
  - Sticky Note (instructional)

- **Node Details:**

  - **When clicking ‘Test workflow’**  
    - Type: Manual Trigger  
    - Role: Starts the workflow manually for testing or execution.  
    - Inputs: None  
    - Outputs: Triggers the next node  
    - Edge Cases: None (manual trigger)  
    - Notes: Entry point for manual execution.

  - **Set URL**  
    - Type: Set  
    - Role: Defines the base domain URL for sitemap access.  
    - Configuration: Assigns a string value to the variable `Domain` (default: `https://phu.io.vn/`).  
    - Expressions: Uses fixed string; no dynamic expressions.  
    - Input: Trigger from manual node  
    - Output: Passes domain URL JSON to next node  
    - Edge Cases: If domain is incorrect or sitemap is not at the standard location (`/sitemap.xml`), subsequent HTTP requests may fail.  
    - Notes: Sticky note advises setting this node or pasting a sitemap URL directly in the next block.

  - **Sticky Note**  
    - Content: "Set website URL at node 1 (or paste sitemap URL at node 2)"  
    - Role: User guidance  
    - Positions visually near Input nodes.

#### 2.2 Sitemap Download and Parsing

- **Overview:**  
  This block downloads the sitemap.xml or sitemap index XML from the given domain or URL, parses the XML content, and splits sitemap indexes into individual sitemap URLs for further processing.

- **Nodes Involved:**  
  - Crawl sitemap  
  - XML  
  - Split Out  
  - Sticky Note (instructional)

- **Node Details:**

  - **Crawl sitemap**  
    - Type: HTTP Request  
    - Role: Downloads sitemap.xml from the domain URL defined previously.  
    - Configuration: URL constructed as `{{ $json.Domain }}sitemap.xml` (e.g., `https://phu.io.vn/sitemap.xml`), timeout set to 10 seconds, response format as string.  
    - Input: Output from Set URL node  
    - Output: Raw XML string of sitemap or sitemap index  
    - Edge Cases: Timeout errors if server is slow; unreachable URL results in HTTP errors.  
    - Notes: Can be replaced by pasting a direct sitemap URL here. Sticky note clarifies this option.

  - **XML**  
    - Type: XML node  
    - Role: Parses the raw XML string into JSON structure for further processing.  
    - Configuration: Default options (no special configuration).  
    - Input: Raw XML string from HTTP Request node  
    - Output: JSON object of sitemap index or sitemap content  
    - Edge Cases: Parsing errors if XML is malformed or empty.

  - **Split Out**  
    - Type: Split Out  
    - Role: Splits the sitemap index into individual sitemap entries (`sitemapindex.sitemap` array).  
    - Configuration: Splits on the field `sitemapindex.sitemap`.  
    - Input: Parsed JSON from XML node  
    - Output: Each sub-sitemap URL as a separate item  
    - Edge Cases: If sitemapindex is missing or sitemap array is empty, no splits occur; workflow may produce no results.

  - **Sticky Note**  
    - Content: "or past sitemap URL at here" (near Crawl sitemap node)  
    - Role: Suggests customization to use a direct sitemap URL instead of constructing from domain.

#### 2.3 Sub-Sitemap Processing

- **Overview:**  
  For each sub-sitemap URL extracted, this block downloads the sub-sitemap, parses its XML, and splits the URL entries so that each URL can be processed or exported individually.

- **Nodes Involved:**  
  - Crawl sitemap 2  
  - XML 2  
  - Split Out 2

- **Node Details:**

  - **Crawl sitemap 2**  
    - Type: HTTP Request  
    - Role: Downloads each sub-sitemap XML using the URL from the previous split.  
    - Configuration: URL set dynamically from `{{ $json.loc }}`, timeout 10 seconds, response format string.  
    - Input: Each sub-sitemap URL from Split Out node  
    - Output: Raw XML string of sub-sitemap content  
    - Edge Cases: Timeout or HTTP errors if sub-sitemap URLs are invalid or server slow.

  - **XML 2**  
    - Type: XML node  
    - Role: Parses each sub-sitemap XML string into JSON.  
    - Configuration: Default options.  
    - Input: Raw XML from Crawl sitemap 2  
    - Output: Parsed JSON containing URLs  
    - Edge Cases: Parsing errors for malformed XML.

  - **Split Out 2**  
    - Type: Split Out  
    - Role: Splits the URL set into individual URL items using the path `urlset.url`.  
    - Configuration: Split on `urlset.url` array.  
    - Input: Parsed JSON from XML 2  
    - Output: Individual URL entries for export or further processing.  
    - Edge Cases: Empty URL sets result in no output items.

#### 2.4 Output Generation

- **Overview:**  
  Converts the extracted URLs into downloadable files or can be replaced with an alternative output like Google Sheets for further SEO analysis.

- **Nodes Involved:**  
  - Convert to File  
  - Sticky Note1

- **Node Details:**

  - **Convert to File**  
    - Type: Convert To File  
    - Role: Converts each URL JSON item into a binary file, enabling download or storage.  
    - Configuration: Binary property name dynamically set from the URL field (`={{ $json.loc }}`).  
    - Input: Individual URLs from Split Out 2  
    - Output: File-ready binary data  
    - Edge Cases: If URL field is missing or improperly formatted, conversion may fail.

  - **Sticky Note1**  
    - Content: "Download file at here (or replace this node = Google sheet node)"  
    - Role: Provides instruction to users on output options and possible customization.

---

### 3. Summary Table

| Node Name              | Node Type           | Functional Role                             | Input Node(s)              | Output Node(s)           | Sticky Note                                                                                  |
|------------------------|---------------------|---------------------------------------------|----------------------------|--------------------------|----------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger      | Triggers the workflow manually               | —                          | Set URL                  |                                                                                              |
| Set URL                | Set                 | Sets base domain URL for sitemap access      | When clicking ‘Test workflow’ | Crawl sitemap            | Set website URL at node 1 (or paste sitemap URL at node 2)                                   |
| Crawl sitemap          | HTTP Request        | Downloads sitemap.xml from domain             | Set URL                    | XML                      | or past sitemap URL at here                                                                  |
| XML                    | XML                 | Parses sitemap XML content                     | Crawl sitemap              | Split Out                 |                                                                                              |
| Split Out              | Split Out           | Splits sitemap index into individual sitemaps | XML                        | Crawl sitemap 2           |                                                                                              |
| Crawl sitemap 2        | HTTP Request        | Downloads each sub-sitemap XML                 | Split Out                  | XML 2                     |                                                                                              |
| XML 2                  | XML                 | Parses sub-sitemap XML                          | Crawl sitemap 2            | Split Out 2               |                                                                                              |
| Split Out 2            | Split Out           | Splits URLs from sub-sitemap                    | XML 2                      | Convert to File           |                                                                                              |
| Convert to File        | Convert To File     | Converts URLs into downloadable file           | Split Out 2                | —                        | Download file at here (or replace this node = Google sheet node)                             |
| Sticky Note            | Sticky Note         | Instruction to set URL or paste sitemap URL   | —                          | —                        | Set website URL at node 1 (or paste sitemap URL at node 2)                                   |
| Sticky Note1           | Sticky Note         | Instruction on output options                   | —                          | —                        | Download file at here (or replace this node = Google sheet node)                             |
| Sticky Note2           | Sticky Note         | FAQ and troubleshooting guidance               | —                          | —                        | # FAQ ... [Full content on errors, usage tips, and compatibility; see section 5 below]      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: `When clicking ‘Test workflow’`  
   - Type: Manual Trigger  
   - Purpose: Start workflow manually.

2. **Create Set Node for Domain URL**  
   - Name: `Set URL`  
   - Type: Set  
   - Add a string field called `Domain` with value e.g., `https://phu.io.vn/`  
   - Connect output of Manual Trigger to this node.

3. **Create HTTP Request Node to Download Sitemap**  
   - Name: `Crawl sitemap`  
   - Type: HTTP Request  
   - URL: Use expression `{{ $json.Domain }}sitemap.xml`  
   - Response Format: String  
   - Timeout: 10 seconds (adjustable)  
   - Connect output of `Set URL` to this node.

4. **Create XML Node to Parse Sitemap Index**  
   - Name: `XML`  
   - Type: XML  
   - Default options  
   - Connect output of `Crawl sitemap` to this node.

5. **Create Split Out Node to Separate Sub-Sitemaps**  
   - Name: `Split Out`  
   - Type: Split Out  
   - Field to split out: `sitemapindex.sitemap`  
   - Connect output of `XML` to this node.

6. **Create HTTP Request Node to Download Each Sub-Sitemap**  
   - Name: `Crawl sitemap 2`  
   - Type: HTTP Request  
   - URL: Expression `{{ $json.loc }}` (each sub-sitemap URL)  
   - Response Format: String  
   - Timeout: 10 seconds  
   - Connect output of `Split Out` to this node.

7. **Create XML Node to Parse Each Sub-Sitemap**  
   - Name: `XML 2`  
   - Type: XML  
   - Default options  
   - Connect output of `Crawl sitemap 2` to this node.

8. **Create Split Out Node to Extract URLs from Sub-Sitemap**  
   - Name: `Split Out 2`  
   - Type: Split Out  
   - Field to split out: `urlset.url`  
   - Connect output of `XML 2` to this node.

9. **Create Convert To File Node for Downloadable Output**  
   - Name: `Convert to File`  
   - Type: Convert To File  
   - Binary property name: Expression `={{ $json.loc }}`  
   - Connect output of `Split Out 2` to this node.

10. *(Optional)* Replace `Convert to File` with Google Sheets node if you prefer URLs exported to a spreadsheet. Configure Google Sheets credentials and map the `loc` field accordingly.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    | Context or Link                                                                                  |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| FAQ covering large sitemap handling, direct sitemap URL usage, timeout adjustments, Google Sheets integration, and compatibility with older n8n versions. Includes advice for scaling n8n instances for large sitemap processing.                                                                                                                                                                                                                                                                                                                                                                                           | See Sticky Note2 content in workflow; important for troubleshooting and customization.           |
| Google Sheets node can replace file conversion node for direct URL export to spreadsheets. This requires setting up Google Sheets credentials in n8n.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | Refer to official n8n Google Sheets node documentation.                                          |
| Timeout for HTTP Request nodes is set to 10 seconds by default; increase if target servers are slow or large sitemaps are expected.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           | Adjustable in HTTP Request node settings under options.                                          |
| The workflow is compatible with n8n version 1.0 and later; check migration guides for deprecated features if using older versions.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             | n8n v1.0 migration guide and official documentation.                                            |
| Workflow designed for SEO specialists and website administrators needing automated URL extraction for audits or analytics.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | Use case context.                                                                               |

---

**Disclaimer:**  
The text provided is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and publicly available.