Exporting Tawk.to Help Center Articles to Google Drive as Markdown Files

https://n8nworkflows.xyz/workflows/exporting-tawk-to-help-center-articles-to-google-drive-as-markdown-files-9785


# Exporting Tawk.to Help Center Articles to Google Drive as Markdown Files

### 1. Workflow Overview

This n8n workflow automates the complete export process of articles from a Tawk.to Help Center website into Google Drive as Markdown (.md) files. It is designed for users who want to back up or migrate their help center content, maintain offline copies, or integrate documentation into other systems.

The workflow is logically divided into the following blocks:

- **1.1 Input & Initialization:** Manual trigger to start the workflow and setting the target website URL.
- **1.2 Category Discovery:** Fetch the homepage HTML, extract all help center categories URLs, and flatten the list.
- **1.3 Article Discovery:** For each category, fetch category page HTML, extract article URLs, and flatten the article list.
- **1.4 Article Content Extraction:** For each article URL, fetch the article HTML content.
- **1.5 Content Conversion & Export:** Convert the HTML article content to Markdown format, check for duplicates on Google Drive, and upload new Markdown files to a specified Google Drive folder.
- **1.6 Duplicate Handling:** Before uploading, verify if the article’s Markdown file already exists in Google Drive to avoid duplication.

---

### 2. Block-by-Block Analysis

#### 1.1 Input & Initialization

- **Overview:** Starts the workflow manually and sets the target Tawk.to Help Center website URL.
- **Nodes Involved:** `When clicking ‘Execute workflow’`, `set-website`, `get-website-home`

##### Node: When clicking ‘Execute workflow’  
- **Type:** Manual Trigger  
- **Role:** Entry point to manually start the workflow.  
- **Configuration:** No parameters, just a manual trigger.  
- **Connections:** Output → `set-website`  
- **Edge cases:** User must manually trigger; no auto-start.  

##### Node: set-website  
- **Type:** Set  
- **Role:** Assign the Help Center website URL to a variable for reuse.  
- **Configuration:** Sets string variable `website` with the URL (e.g., "https://meurastreio.tawk.help").  
- **Connections:** Input ← `When clicking ‘Execute workflow’`; Output → `get-website-home`  
- **Edge cases:** Must be updated manually to target the correct Help Center domain.  

##### Node: get-website-home  
- **Type:** HTTP Request  
- **Role:** Fetch the homepage HTML content of the Help Center.  
- **Configuration:** Uses URL from `website` variable; no auth or special headers.  
- **Connections:** Input ← `set-website`; Output → `find-categories`  
- **Edge cases:** HTTP errors (404, 500), network timeouts, or invalid URL can cause failure.  

---

#### 1.2 Category Discovery

- **Overview:** Extracts all category URLs from the homepage HTML and prepares the list for further processing.  
- **Nodes Involved:** `find-categories`, `flat-categories`, `website-category`

##### Node: find-categories  
- **Type:** HTML Extract  
- **Role:** Parses homepage HTML to find all category links.  
- **Configuration:** CSS selector `.category-block > a`, extracts `href` attributes as array.  
- **Connections:** Input ← `get-website-home`; Output → `flat-categories`  
- **Edge cases:** If HTML structure changes or selector is incorrect, no categories may be extracted.  

##### Node: flat-categories  
- **Type:** Code  
- **Role:** Transforms extracted categories array into individual category objects for iteration.  
- **Configuration:** JavaScript code: `return $input.first().json.categories.map(category => ({category}))`  
- **Connections:** Input ← `find-categories`; Output → `website-category`  
- **Edge cases:** Empty or malformed categories array leads to no further processing.  

##### Node: website-category  
- **Type:** HTTP Request  
- **Role:** Fetch the HTML content of each category page using the category URL.  
- **Configuration:** URL constructed as `website` base URL + category path from input.  
- **Connections:** Input ← `flat-categories`; Output → `find-category-articles`  
- **Edge cases:** HTTP errors, invalid category URLs.  

---

#### 1.3 Article Discovery

- **Overview:** Extracts article URLs from each category page and flattens the list for article-level iteration.  
- **Nodes Involved:** `find-category-articles`, `flat-articles`, `Loop Over Items`

##### Node: find-category-articles  
- **Type:** HTML Extract  
- **Role:** Extracts article links from category pages.  
- **Configuration:** CSS selector `.article-block > a`, extracts `href` attributes as array.  
- **Connections:** Input ← `website-category`; Output → `flat-articles`  
- **Edge cases:** Changes in HTML structure or selectors may cause failure to find articles.  

##### Node: flat-articles  
- **Type:** Code  
- **Role:** Flattens article URLs into individual items with metadata for processing.  
- **Configuration:** JS code iterates all input items, constructs article names (slugified paths joined with dashes) and full article URLs.  
- **Key expressions:** Uses `$('set-website').first().json.website` for base URL.  
- **Connections:** Input ← `find-category-articles`; Output → `Loop Over Items`  
- **Edge cases:** Empty article lists or malformed URLs can halt further processing.  

##### Node: Loop Over Items  
- **Type:** SplitInBatches  
- **Role:** Processes articles one by one to manage load and avoid request flooding.  
- **Configuration:** Default batch size (not explicitly set)  
- **Connections:** Input ← `flat-articles`; Outputs: main 0 → (no connection), main 1 → `Search files and folders`  
- **Edge cases:** Large batch sizes can cause timeouts; no batch size set can cause performance issues.  

---

#### 1.4 Article Content Extraction

- **Overview:** For each article URL, fetch the HTML content and extract the main article content.  
- **Nodes Involved:** `Search files and folders`, `is-duplicated`, `website-article`, `extract-article-content`

##### Node: Search files and folders  
- **Type:** Google Drive (Search)  
- **Role:** Searches Google Drive for existing Markdown files with the article’s name to avoid duplicates.  
- **Configuration:** Query string set to `${articleName}.md`, limits to 1 result, uses Service Account credentials.  
- **Connections:** Input ← `Loop Over Items` (batch output); Output → `is-duplicated`  
- **Edge cases:** Google API quota limits, auth failures, or incorrect folder scope can cause failures.  

##### Node: is-duplicated  
- **Type:** If  
- **Role:** Decides if the article file already exists in Drive.  
- **Configuration:** Checks if the filename exists and matches the expected name; condition logic using expression comparisons.  
- **Connections:** Input ← `Search files and folders`; Outputs: True (duplicate) → `Loop Over Items` (to skip), False (new file) → `website-article`  
- **Edge cases:** Expression errors or logic misconfigurations may cause incorrect skipping or duplication.  

##### Node: website-article  
- **Type:** HTTP Request  
- **Role:** Fetches the full article HTML page from the URL.  
- **Configuration:** URL from current batch item `articleUri`.  
- **Connections:** Input ← `is-duplicated` (False branch); Output → `extract-article-content`  
- **Edge cases:** HTTP errors, network issues, or unavailable articles.  

##### Node: extract-article-content  
- **Type:** HTML Extract  
- **Role:** Extracts the main article content HTML from the fetched page.  
- **Configuration:** CSS selector `#article-wrapper` with HTML return type.  
- **Connections:** Input ← `website-article`; Output → `convert-to-markdown`  
- **Edge cases:** Selector changes or missing content causes empty extraction.  

---

#### 1.5 Content Conversion & Export

- **Overview:** Converts HTML article content to Markdown, then creates Markdown files in Google Drive.  
- **Nodes Involved:** `convert-to-markdown`, `Create file from text`

##### Node: convert-to-markdown  
- **Type:** Markdown  
- **Role:** Converts extracted HTML content into Markdown format, preserving inline base64 images.  
- **Configuration:** Input HTML from `article` field; option to keep data images enabled.  
- **Connections:** Input ← `extract-article-content`; Output → `Create file from text`  
- **Edge cases:** Conversion inaccuracies if HTML is malformed; large embedded images increase file size.  

##### Node: Create file from text  
- **Type:** Google Drive (Create file)  
- **Role:** Uploads the converted Markdown content as a file to a Google Drive folder.  
- **Configuration:**  
  - Filename: `${name}.md` from current article item.  
  - Content: Markdown content from conversion node.  
  - Folder ID: specific Google Drive folder ("Meu Rastreio - Tutoriais").  
  - Authentication: Google Service Account credentials.  
- **Connections:** Input ← `convert-to-markdown`; Output → `Loop Over Items` (to continue processing next article)  
- **Edge cases:** Google API quota limits, permission errors, filename conflicts.

---

#### 1.6 Duplicate Handling

- **Overview:** Prevents duplicate uploads by checking existing files on Google Drive before processing the article content.  
- **Nodes Involved:** `Search files and folders`, `is-duplicated`

(This was covered in previous blocks but emphasized here as a dedicated logical step.)

---

### 3. Summary Table

| Node Name                   | Node Type           | Functional Role                               | Input Node(s)                  | Output Node(s)                      | Sticky Note                                                                                      |
|-----------------------------|---------------------|----------------------------------------------|-------------------------------|-----------------------------------|------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger       | Manual start of the workflow                  | —                             | set-website                       |                                                                                                |
| set-website                 | Set                 | Define target Help Center website URL         | When clicking ‘Execute workflow’ | get-website-home                  | # Config: Put your website to extract here                                                    |
| get-website-home            | HTTP Request        | Fetch homepage HTML content                    | set-website                   | find-categories                   |                                                                                                |
| find-categories             | HTML Extract        | Extract category URLs from homepage            | get-website-home              | flat-categories                  | # Search Process: 1) Here we'll find all articles on your website; 2) Extract all urls        |
| flat-categories             | Code                | Flatten categories array                        | find-categories               | website-category                 | # Search Process: 1) Here we'll find all articles on your website; 2) Extract all urls        |
| website-category            | HTTP Request        | Fetch category page HTML                        | flat-categories               | find-category-articles           | # Search Process: 1) Here we'll find all articles on your website; 2) Extract all urls        |
| find-category-articles      | HTML Extract        | Extract article URLs from category pages       | website-category              | flat-articles                   | # Search Process: 1) Here we'll find all articles on your website; 2) Extract all urls        |
| flat-articles               | Code                | Flatten article URLs and create metadata       | find-category-articles        | Loop Over Items                 | # Search Process: 1) Here we'll find all articles on your website; 2) Extract all urls        |
| Loop Over Items             | SplitInBatches      | Process articles in batches                     | flat-articles                 | Search files and folders (main 1); no connection (main 0) |                                                                                                |
| Search files and folders    | Google Drive (Search) | Check if article Markdown exists in Drive      | Loop Over Items               | is-duplicated                   | # Extraction: 1) Configure Google Drive Access; 2) Extract HTML; 3) Convert to Markdown; 4) Upload to Google Drive |
| is-duplicated               | If                  | Determine if article file already exists       | Search files and folders      | Loop Over Items (duplicate), website-article (new) | # Extraction: 1) Configure Google Drive Access; 2) Extract HTML; 3) Convert to Markdown; 4) Upload to Google Drive |
| website-article             | HTTP Request        | Fetch article HTML content                      | is-duplicated (False)         | extract-article-content          | # Extraction: 1) Configure Google Drive Access; 2) Extract HTML; 3) Convert to Markdown; 4) Upload to Google Drive |
| extract-article-content     | HTML Extract        | Extract main article HTML content               | website-article               | convert-to-markdown             | # Extraction: 1) Configure Google Drive Access; 2) Extract HTML; 3) Convert to Markdown; 4) Upload to Google Drive |
| convert-to-markdown         | Markdown            | Convert HTML article content to Markdown       | extract-article-content       | Create file from text           | # Extraction: 1) Configure Google Drive Access; 2) Extract HTML; 3) Convert to Markdown; 4) Upload to Google Drive |
| Create file from text       | Google Drive (Create) | Upload Markdown file to Google Drive            | convert-to-markdown           | Loop Over Items                 | # Extraction: 1) Configure Google Drive Access; 2) Extract HTML; 3) Convert to Markdown; 4) Upload to Google Drive |
| Config                      | Sticky Note         | Configuration instructions                      | —                             | —                               | # Config: Put your website to extract here                                                    |
| Search                     | Sticky Note         | Description of search process                    | —                             | —                               | # Search Process: 1) Here we'll find all articles on your website; 2) Extract all urls        |
| Extraction                 | Sticky Note         | Description of extraction and export process    | —                             | —                               | # Extraction: 1) Configure Google Drive Access; 2) Extract HTML; 3) Convert to Markdown; 4) Upload to Google Drive |
| Workflow Information       | Sticky Note         | High-level workflow explanation                  | —                             | —                               | # ⚙️ How the Workflow Works: The Tawk Help Export workflow in n8n automates the export ...    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Purpose: Start workflow manually.  

2. **Add Set Node for Website URL**  
   - Type: Set  
   - Add string parameter `website` with your Tawk.to Help Center base URL (e.g., https://meurastreio.tawk.help).  
   - Connect output of Manual Trigger → this node.

3. **Add HTTP Request Node to Fetch Homepage**  
   - Type: HTTP Request  
   - URL: Expression referencing `website` variable (`{{$json.website}}`)  
   - No authentication required.  
   - Connect output of Set Node → this node.

4. **Add HTML Extract Node to Find Categories**  
   - Type: HTML Extract  
   - Operation: Extract HTML Content  
   - Extraction Values:  
     - Key: `categories`  
     - CSS Selector: `.category-block > a`  
     - Return Attribute: `href`  
     - Return Array: True  
   - Connect output of HTTP Request (homepage) → this node.

5. **Add Code Node to Flatten Categories**  
   - Type: Code  
   - JavaScript code:  
     ```js
     return $input.first().json.categories.map(category => ({category}))
     ```  
   - Connect output of HTML Extract (categories) → this node.

6. **Add HTTP Request Node for Each Category Page**  
   - Type: HTTP Request  
   - URL: Expression: `{{$node["set-website"].json.website}}{{$json.category}}`  
   - Connect output of Code Node (flatten categories) → this node.

7. **Add HTML Extract Node to Find Articles in Category**  
   - Type: HTML Extract  
   - Operation: Extract HTML Content  
   - Extraction Values:  
     - Key: `articles`  
     - CSS Selector: `.article-block > a`  
     - Return Attribute: `href`  
     - Return Array: True  
   - Connect output of HTTP Request (category page) → this node.

8. **Add Code Node to Flatten Articles**  
   - Type: Code  
   - JavaScript code:  
     ```js
     const articles = [];
     for (const item of $input.all()) {
       for (const article of item.json.articles) {
         articles.push({
           json: {
             name: article.split('/').filter(Boolean).join('-'),
             articleUri: $node['set-website'].json.website + article
           },
           pairedItem: { item: item.binary }
         });
       }
     }
     return articles;
     ```  
   - Connect output of HTML Extract (articles) → this node.

9. **Add SplitInBatches Node**  
   - Type: SplitInBatches  
   - Purpose: Process articles one by one to avoid overload.  
   - Connect output of Code Node (flatten articles) → this node.

10. **Add Google Drive Search Node**  
    - Type: Google Drive (Search)  
    - Resource: fileFolder  
    - Query String: `{{$json.name}}.md`  
    - Limit: 1  
    - Authentication: Google Service Account (configure credentials)  
    - Connect output of SplitInBatches → this node.

11. **Add If Node to Check for Duplicates**  
    - Type: If  
    - Conditions:  
      - Check if file with name exists (`{{$json.name}}.md`)  
      - Use appropriate expression to detect if Google Drive search returned results matching current article name.  
    - Connect output of Google Drive Search → this node.

12. **From If Node, connect False branch (no duplicate) to HTTP Request for Article**  
    - Type: HTTP Request  
    - URL: `{{$json.articleUri}}`  
    - Connect False output of If → this node.

13. **Add HTML Extract Node to Extract Article Content**  
    - Type: HTML Extract  
    - Extraction Values:  
      - Key: `article`  
      - CSS Selector: `#article-wrapper`  
      - Return Value: HTML  
    - Connect output of HTTP Request (article) → this node.

14. **Add Markdown Node to Convert HTML to Markdown**  
    - Type: Markdown  
    - Input: Expression `{{$json.article}}`  
    - Option: Keep data images enabled.  
    - Connect output of HTML Extract (article content) → this node.

15. **Add Google Drive Create File Node**  
    - Type: Google Drive (Create File)  
    - Filename: Expression `{{$node["Loop Over Items"].json.name}}.md`  
    - Content: Expression `{{$json.data}}` (markdown content)  
    - Folder ID: Target Google Drive folder ID (e.g., "1VbQVYn33euu9WFDGTFUr965hmH208uJE")  
    - Authentication: Google Service Account (same credentials as Search)  
    - Connect output of Markdown node → this node.

16. **Connect output of Google Drive Create File → SplitInBatches Node**  
    - This loops back to process next batch item.

17. **From If Node, connect True branch (duplicate found) → SplitInBatches Node**  
    - Skips duplicate and continues with next article.

18. **Add Sticky Notes (optional)**  
    - Add explanatory sticky notes for configuration, search process, extraction, and overall workflow info as per user preference.

19. **Credential Setup**  
    - Configure Google Service Account credentials with Drive API access.  
    - No authentication required for HTTP requests to public website.

---

### 5. General Notes & Resources

| Note Content                                                                                                                          | Context or Link                                                                                  |
|-------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| The workflow automates backup of Tawk.to Help Center articles, saving them as Markdown files on Google Drive for offline use.       | Workflow description                                                                              |
| Google Drive folder for uploads is set to “Meu Rastreio - Tutoriais” with folder ID `1VbQVYn33euu9WFDGTFUr965hmH208uJE`.            | Modify folder ID in Create File node to target different folder                                  |
| HTML selectors (e.g., `.category-block > a`, `.article-block > a`, `#article-wrapper`) are critical and may require updating if Tawk.to changes their site structure. | CSS selectors in HTML Extract nodes                                                              |
| Uses Google Drive Service Account credentials; ensure proper permissions and Drive API enabled.                                       | Credential configuration                                                                         |
| Markdown node option “keepDataImages” is enabled to preserve embedded base64 images in articles.                                     | Markdown conversion settings                                                                     |
| Manual workflow start provides control but can be automated via schedule triggers if desired.                                        | Workflow trigger                                                                                |
| For large help centers, consider setting batch size in SplitInBatches to avoid API rate limits or timeouts.                         | Performance tuning                                                                              |

---

**Disclaimer:** The provided text is derived exclusively from an automated workflow built with n8n, an integration and automation tool. This processing strictly complies with current content policies and includes no illegal, offensive, or protected elements. All data handled is lawful and publicly accessible.