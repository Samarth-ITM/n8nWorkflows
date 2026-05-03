💡🌐 Essential Multipage Website Scraper with Jina.ai

https://n8nworkflows.xyz/workflows/-----essential-multipage-website-scraper-with-jina-ai-2957


# 💡🌐 Essential Multipage Website Scraper with Jina.ai

### 1. Workflow Overview

This workflow automates the scraping of multiple web pages from a website’s sitemap using Jina.ai’s web scraping service, then saves each page’s content as a markdown document in Google Drive. It is designed for responsible, batch-processed multi-page scraping with filtering and rate limiting.

**Target Use Cases:**  
- Extracting structured content from large websites with sitemaps  
- Archiving or analyzing website content in markdown format  
- Automating content ingestion pipelines without requiring API keys for scraping  

**Logical Blocks:**  
- **1.1 Input Reception & Sitemap Processing:** Accepts a sitemap URL, fetches and parses it into individual page URLs, then filters URLs based on user-defined criteria.  
- **1.2 Pagination & Batch Control:** Limits the number of pages processed per run and splits the list into manageable batches.  
- **1.3 Web Scraping via Jina.ai:** Calls Jina.ai’s scraper for each URL to extract page content in markdown format and page title.  
- **1.4 Content Extraction & Formatting:** Parses the raw scraper output to separate the page title and markdown content.  
- **1.5 Storage Integration:** Creates Google Drive documents named by URL and title, storing the markdown content.  
- **1.6 Rate Limiting & Looping:** Waits between batches to avoid server overload and loops over all batches until complete.  

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Sitemap Processing

- **Overview:**  
  This block initializes the workflow with a sitemap URL, fetches the sitemap XML, converts it to JSON, extracts individual URLs, and filters them based on topic or page criteria.

- **Nodes Involved:**  
  - When clicking ‘Test workflow’ (Manual Trigger)  
  - Set Website URL  
  - Get List of Website URLs (HTTP Request)  
  - Convert to JSON (XML to JSON)  
  - Create List of Website URLs (Split Out)  
  - Filter By Topics or Pages (Filter)  
  - Limit (Limit)

- **Node Details:**  

  1. **When clicking ‘Test workflow’**  
     - Type: Manual Trigger  
     - Role: Entry point to start the workflow manually  
     - Config: No parameters  
     - Inputs: None  
     - Outputs: Connects to "Set Website URL"  
     - Edge cases: None  

  2. **Set Website URL**  
     - Type: Set  
     - Role: Defines the sitemap URL input variable `sitemap_url`  
     - Config: Default value set to `https://ai.pydantic.dev/sitemap.xml`  
     - Inputs: From manual trigger  
     - Outputs: To HTTP Request node  
     - Edge cases: Invalid or unreachable sitemap URL may cause HTTP request failure  

  3. **Get List of Website URLs**  
     - Type: HTTP Request  
     - Role: Fetches sitemap XML from the URL in `sitemap_url`  
     - Config: URL set dynamically from `{{$json.sitemap_url}}`  
     - Inputs: From "Set Website URL"  
     - Outputs: To "Convert to JSON"  
     - Edge cases: Network errors, invalid sitemap format, HTTP errors  

  4. **Convert to JSON**  
     - Type: XML  
     - Role: Converts sitemap XML to JSON for easier processing  
     - Config: Default XML to JSON conversion  
     - Inputs: From HTTP Request  
     - Outputs: To "Create List of Website URLs"  
     - Edge cases: Malformed XML could cause parsing errors  

  5. **Create List of Website URLs**  
     - Type: Split Out  
     - Role: Extracts array of URLs from JSON path `urlset.url`  
     - Config: Field to split out: `urlset.url`  
     - Inputs: From XML node  
     - Outputs: To "Filter By Topics or Pages"  
     - Edge cases: Empty or missing `urlset.url` array  

  6. **Filter By Topics or Pages**  
     - Type: Filter  
     - Role: Filters URLs based on conditions to target specific topics or pages  
     - Config: Conditions include:  
       - URL equals `https://ai.pydantic.dev/`  
       - URL contains "agent" (case insensitive)  
       - URL contains "tool" (case insensitive)  
     - Inputs: From split out URLs  
     - Outputs: To "Limit" node  
     - Edge cases: No URLs matching filter results in empty output  

  7. **Limit**  
     - Type: Limit  
     - Role: Restricts the number of URLs processed per run (default 20)  
     - Config: Max items = 20  
     - Inputs: From filter node  
     - Outputs: To "Loop Over Items"  
     - Edge cases: Limit too low may skip desired URLs; too high may overload downstream nodes  

---

#### 2.2 Pagination & Batch Control

- **Overview:**  
  Splits the filtered list of URLs into batches for sequential processing to manage load and rate limits.

- **Nodes Involved:**  
  - Loop Over Items (Split In Batches)  
  - Wait  

- **Node Details:**  

  1. **Loop Over Items**  
     - Type: Split In Batches  
     - Role: Processes URLs one batch at a time (default batch size not explicitly set, defaults to 1)  
     - Config: Default options, no batch size override  
     - Inputs: From "Limit" node  
     - Outputs: Two outputs:  
       - Output 1: Empty (used for looping)  
       - Output 2: To "Jina.ai Web Scraper" for processing each URL  
     - Edge cases: Batch size too small slows workflow; too large risks rate limits  

  2. **Wait**  
     - Type: Wait  
     - Role: Introduces delay between batches to prevent server overload  
     - Config: Default wait time (not explicitly set, so default n8n wait applies)  
     - Inputs: From "Save Webpage Contents to Google Drive"  
     - Outputs: Loops back to "Loop Over Items" to process next batch  
     - Edge cases: No wait or too short wait risks IP blocking; too long wait slows processing  

---

#### 2.3 Web Scraping via Jina.ai

- **Overview:**  
  For each URL batch item, calls Jina.ai’s web scraper API to extract webpage content.

- **Nodes Involved:**  
  - Jina.ai Web Scraper (HTTP Request)  

- **Node Details:**  

  1. **Jina.ai Web Scraper**  
     - Type: HTTP Request  
     - Role: Calls Jina.ai scraper endpoint with the URL to scrape content  
     - Config: URL dynamically constructed as `https://r.jina.ai/{{ $json.loc }}` where `loc` is the page URL  
     - Inputs: From "Loop Over Items" (batch output)  
     - Outputs: To "Extract Title & Markdown Content"  
     - Edge cases: Network errors, invalid URLs, scraper service downtime, unexpected response format  
     - Notes: No API key required, simplifying setup  

---

#### 2.4 Content Extraction & Formatting

- **Overview:**  
  Parses the raw text response from Jina.ai to extract the page title and markdown content separately.

- **Nodes Involved:**  
  - Extract Title & Markdown Content (Code)

- **Node Details:**  

  1. **Extract Title & Markdown Content**  
     - Type: Code (JavaScript)  
     - Role: Uses regex to parse the scraper’s text output into two fields: `title` and `markdown`  
     - Config:  
       - Extracts title from line starting with `Title:`  
       - Extracts markdown content following `Markdown Content:`  
     - Inputs: From "Jina.ai Web Scraper"  
     - Outputs: To "Save Webpage Contents to Google Drive"  
     - Edge cases: Unexpected text format breaks regex, missing title or markdown results in empty fields  

---

#### 2.5 Storage Integration

- **Overview:**  
  Saves each scraped page’s markdown content as a Google Drive document, named by URL and page title.

- **Nodes Involved:**  
  - Save Webpage Contents to Google Drive  

- **Node Details:**  

  1. **Save Webpage Contents to Google Drive**  
     - Type: Google Drive  
     - Role: Creates a new text document in Google Drive with scraped content  
     - Config:  
       - Document name: `{{ $('Loop Over Items').item.json.loc }} - {{ $json.title }}` (combines URL and extracted title)  
       - Content: Markdown text from previous node  
       - Drive: "My Drive"  
       - Folder: Root folder  
       - Operation: Create from text  
     - Inputs: From "Extract Title & Markdown Content"  
     - Outputs: To "Wait" node for rate limiting  
     - Credentials: Requires configured Google Drive OAuth2 credentials  
     - Edge cases: Google Drive API errors, permission issues, naming conflicts  

---

#### 2.6 Rate Limiting & Looping

- **Overview:**  
  Implements a wait period after saving each document before processing the next batch, looping until all batches are processed.

- **Nodes Involved:**  
  - Wait  
  - Loop Over Items (loop back)

- **Node Details:**  

  1. **Wait**  
     - See details in 2.2  

  2. **Loop Over Items**  
     - See details in 2.2  

---

### 3. Summary Table

| Node Name                      | Node Type           | Functional Role                              | Input Node(s)                   | Output Node(s)                     | Sticky Note                                                                                  |
|-------------------------------|---------------------|----------------------------------------------|--------------------------------|-----------------------------------|----------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’  | Manual Trigger      | Entry point to start workflow manually       | None                           | Set Website URL                   | ## 👍Try Me!                                                                                |
| Set Website URL                | Set                 | Defines sitemap URL input                      | When clicking ‘Test workflow’  | Get List of Website URLs          | ## 👇Add Website Sitemap URL                                                                |
| Get List of Website URLs       | HTTP Request        | Fetches sitemap XML from URL                   | Set Website URL                | Convert to JSON                   |                                                                                              |
| Convert to JSON                | XML                 | Converts sitemap XML to JSON                    | Get List of Website URLs       | Create List of Website URLs       |                                                                                              |
| Create List of Website URLs    | Split Out           | Extracts array of URLs from sitemap JSON       | Convert to JSON                | Filter By Topics or Pages         |                                                                                              |
| Filter By Topics or Pages      | Filter              | Filters URLs by topic/page criteria             | Create List of Website URLs    | Limit                           |                                                                                              |
| Limit                         | Limit               | Limits number of URLs processed per run         | Filter By Topics or Pages      | Loop Over Items                  |                                                                                              |
| Loop Over Items               | Split In Batches    | Processes URLs in batches                        | Limit                         | Jina.ai Web Scraper, Loop Over Items (loop) |                                                                                              |
| Jina.ai Web Scraper            | HTTP Request        | Scrapes webpage content via Jina.ai             | Loop Over Items               | Extract Title & Markdown Content  | ## Jina.ai Web Scraper\n### No API Key Required                                             |
| Extract Title & Markdown Content| Code                | Parses scraper output into title and markdown  | Jina.ai Web Scraper            | Save Webpage Contents to Google Drive |                                                                                              |
| Save Webpage Contents to Google Drive | Google Drive        | Saves markdown content as Google Drive document | Extract Title & Markdown Content | Wait                            |                                                                                              |
| Wait                          | Wait                | Rate limits processing between batches          | Save Webpage Contents to Google Drive | Loop Over Items               |                                                                                              |
| Sticky Note                   | Sticky Note         | Informational note                              | None                           | None                            | ## Jina.ai Web Scraper\n### No API Key Required                                             |
| Sticky Note1                  | Sticky Note         | Workflow title and usage note                   | None                           | None                            | # 💡🌐 Essential Multipage Website Scraper with Jina.ai\n## Scrape entire websites with this workflow\n**Use responsibly and follow local rules and regulations** |
| Sticky Note2                  | Sticky Note         | Encouragement to try workflow                    | None                           | None                            | ## 👍Try Me!                                                                                |
| Sticky Note3                  | Sticky Note         | Instruction to add sitemap URL                   | None                           | None                            | ## 👇Add Website Sitemap URL                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: `When clicking ‘Test workflow’`  
   - Purpose: Manual start of workflow  
   - No parameters  

2. **Create Set Node**  
   - Name: `Set Website URL`  
   - Add field: `sitemap_url` (string)  
   - Default value: `https://ai.pydantic.dev/sitemap.xml`  
   - Connect output of manual trigger to this node  

3. **Create HTTP Request Node**  
   - Name: `Get List of Website URLs`  
   - Method: GET  
   - URL: Expression `{{$json.sitemap_url}}`  
   - Connect output of Set Website URL to this node  

4. **Create XML Node**  
   - Name: `Convert to JSON`  
   - Operation: XML to JSON conversion (default)  
   - Connect output of HTTP Request to this node  

5. **Create Split Out Node**  
   - Name: `Create List of Website URLs`  
   - Field to split out: `urlset.url`  
   - Connect output of XML node to this node  

6. **Create Filter Node**  
   - Name: `Filter By Topics or Pages`  
   - Conditions (OR):  
     - `{{$json.loc}}` equals `https://ai.pydantic.dev/`  
     - `{{$json.loc.toLowerCase()}}` contains `agent`  
     - `{{$json.loc.toLowerCase()}}` contains `tool`  
   - Connect output of Split Out node to this node  

7. **Create Limit Node**  
   - Name: `Limit`  
   - Max Items: 20  
   - Connect output of Filter node to this node  

8. **Create Split In Batches Node**  
   - Name: `Loop Over Items`  
   - Default batch size (1) or adjust as needed  
   - Connect output of Limit node to this node  

9. **Create HTTP Request Node**  
   - Name: `Jina.ai Web Scraper`  
   - Method: GET  
   - URL: Expression `https://r.jina.ai/{{ $json.loc }}`  
   - Connect batch output of Loop Over Items to this node  

10. **Create Code Node**  
    - Name: `Extract Title & Markdown Content`  
    - Language: JavaScript  
    - Code:  
      ```javascript
      const data = $input.first().json.data;
      const titleRegex = /^Title:\s*(.+)$/m;
      const markdownRegex = /Markdown Content:\n([\s\S]+)/;
      const titleMatch = data.match(titleRegex);
      const title = titleMatch ? titleMatch[1].trim() : '';
      const markdownMatch = data.match(markdownRegex);
      const markdown = markdownMatch ? markdownMatch[1].trim() : '';
      return { title, markdown };
      ```  
    - Connect output of Jina.ai Web Scraper to this node  

11. **Create Google Drive Node**  
    - Name: `Save Webpage Contents to Google Drive`  
    - Operation: Create from text  
    - Name: Expression `{{ $('Loop Over Items').item.json.loc }} - {{ $json.title }}`  
    - Content: Expression `{{ $json.markdown }}`  
    - Drive: My Drive  
    - Folder: Root folder  
    - Credentials: Configure Google Drive OAuth2 credentials  
    - Connect output of Code node to this node  

12. **Create Wait Node**  
    - Name: `Wait`  
    - Default wait time or configure delay as needed (e.g., 1-5 seconds)  
    - Connect output of Google Drive node to this node  

13. **Loop Back**  
    - Connect output of Wait node back to the Loop Over Items node to process next batch  

14. **Add Sticky Notes** (optional for documentation)  
    - Add notes for Jina.ai Web Scraper (no API key required)  
    - Add workflow title and usage instructions  
    - Add encouragement and input instructions  

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                                  |
|-------------------------------------------------------------------------------------------------|-------------------------------------------------|
| **Jina.ai Web Scraper - No API Key Required**                                                   | Sticky note on scraper node                      |
| **💡🌐 Essential Multipage Website Scraper with Jina.ai**<br>Scrape entire websites responsibly. | Workflow title sticky note                        |
| **Use responsibly and follow local rules and regulations**                                      | Workflow description and sticky notes            |
| **Try Me!**                                                                                    | Encouragement sticky note near manual trigger    |
| [Jina.ai official website](https://jina.ai)                                                    | Reference for scraper service                      |

---

This documentation fully describes the workflow’s structure, node configurations, and logic flow, enabling reproduction, modification, and troubleshooting by advanced users or automation agents.