ScrapingBee and Google Sheets Integration Template

https://n8nworkflows.xyz/workflows/scrapingbee-and-google-sheets-integration-template-7927


# ScrapingBee and Google Sheets Integration Template

### 1. Workflow Overview

This workflow, titled **Sitemap Link Extractor**, automates the process of extracting links from a website's sitemap and related files, then appends these links to a Google Sheet for further use. It is designed for users who want to gather URLs from sitemaps efficiently, including handling compressed sitemap files. The workflow includes logical blocks for receiving input, scraping site files, processing and extracting links, decompressing files if needed, and storing results.

Logical blocks:

- **1.1 Input Reception:** Receives a webhook request with a domain query parameter.
- **1.2 Scraping Robots.txt:** Attempts to obtain sitemap links from the robots.txt file.
- **1.3 Scraping Sitemap.xml:** If robots.txt does not provide sitemap links, tries sitemap.xml.
- **1.4 Binary Content Handling:** Checks if scraped content is binary, decompresses if `.gz`, normalizes binary data.
- **1.5 Link Extraction:** Separates XML sitemap links from non-XML links.
- **1.6 Recursive Scraping of XML Links:** Scrapes additional XML links recursively.
- **1.7 Storing Links:** Appends extracted non-XML links to a Google Sheet for record-keeping.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** Receives the domain to scrape via a webhook with a query parameter.
- **Nodes Involved:**  
  - Domain to scrape  
  - Sticky Note (Input instructions)

- **Node Details:**  
  - **Domain to scrape**  
    - Type: Webhook  
    - Role: Entry point for domain input.  
    - Config: Listens on path `1da30868-fbca-4e8e-8580-485afb3fd956`. Requires query parameter `domain`.  
    - Inputs: None (trigger)  
    - Outputs: Scrape robots.txt file node  
    - Edge cases: Missing domain parameter or malformed requests may cause failure or no data.

  - **Sticky Note (Input instructions)**  
    - Type: Sticky Note  
    - Role: Provides usage instructions for the webhook input.  
    - Content: Instructs to send a webhook request with `domain` query parameter, example provided.

#### 2.2 Scraping Robots.txt

- **Overview:** Scrapes the site's robots.txt file to find sitemap links, which is the common place for sitemap declarations.
- **Nodes Involved:**  
  - Scrape robots.txt file  
  - If sitemap links are found  
  - Sticky Note (about scraping robots.txt)

- **Node Details:**  
  - **Scrape robots.txt file**  
    - Type: ScrapingBee API node  
    - Role: Request robots.txt content from the domain.  
    - Config: URL constructed as `https://{{domain}}/robots.txt`, JS rendering disabled.  
    - Credentials: Uses ScrapingBee API credentials.  
    - Outputs: Condition node to check for sitemap links or fallback to sitemap.xml.  
    - Edge cases: Robots.txt missing or inaccessible; network or API errors handled with retry and continue on error.

  - **If sitemap links are found**  
    - Type: IF node  
    - Role: Checks if the robots.txt response contains "Sitemap:" string indicating presence of sitemap links.  
    - Inputs: Scrape robots.txt output  
    - Outputs: If true, proceeds to extract XML links; else scrape sitemap.xml.  
    - Edge cases: False negatives if sitemap links are formatted unusually.

  - **Sticky Note (Scrape Robots.txt)**  
    - Role: Explains rationale for scraping robots.txt first to find sitemap links.

#### 2.3 Scraping Sitemap.xml

- **Overview:** If robots.txt lacks sitemap links, attempts scraping the conventional sitemap.xml file directly.
- **Nodes Involved:**  
  - Scrape sitemap.xml file  
  - If it's a binary file  
  - Sticky Note (about scraping sitemap.xml)

- **Node Details:**  
  - **Scrape sitemap.xml file**  
    - Type: ScrapingBee API node  
    - Role: Request sitemap.xml content from the domain.  
    - Config: URL constructed as `https://{{domain}}/sitemap.xml`, JS rendering disabled.  
    - Credentials: ScrapingBee API credentials.  
    - Outputs: IF node to check if content is binary.  
    - Edge cases: sitemap.xml missing or inaccessible.  

  - **If it's a binary file**  
    - Type: IF node  
    - Role: Checks if the response data is binary (important because compressed sitemaps are binary).  
    - Inputs: Output from sitemap.xml scraping or recursive XML scrape.  
    - Outputs: Branches to decompress `.gz` files or directly to link extraction.  
    - Edge cases: Misclassification of binary vs text may cause errors in processing.

  - **Sticky Note (Scrape Sitemap.xml)**  
    - Role: Explains fallback strategy to scrape sitemap.xml if no sitemap links in robots.txt.

#### 2.4 Binary Content Handling and Decompression

- **Overview:** Handles binary payloads, decompresses `.gz` files and normalizes binary key naming for further processing.
- **Nodes Involved:**  
  - If it's a .gz file  
  - Decompress .gz file  
  - Store the file to data key for easy handling  
  - Load the xml file as JSON  
  - Sticky Note (about .gz decompression and key renaming)

- **Node Details:**  
  - **If it's a .gz file**  
    - Type: IF node  
    - Role: Checks if the binary file extension is `.gz`.  
    - Inputs: IF node output (binary check)  
    - Outputs: To decompress or directly to load XML file as JSON.  
    - Edge cases: Incorrect extension or missing binary data may cause failure.

  - **Decompress .gz file**  
    - Type: Compression node  
    - Role: Decompresses `.gz` binary files.  
    - Inputs: From `.gz` check node.  
    - Outputs: To code node for renaming binary key.  
    - Edge cases: Corrupted `.gz` file, decompression failure.  

  - **Store the file to data key for easy handling**  
    - Type: Code node  
    - Role: Renames binary property from `file_0` to `data` for consistency.  
    - Code: Moves `item.binary.file_0` to `item.binary.data`.  
    - Inputs: Decompress output  
    - Outputs: Load XML file as JSON node

  - **Load the xml file as JSON**  
    - Type: Extract From File node (XML extraction)  
    - Role: Parses the binary XML data into JSON for link extraction.  
    - Inputs: Code node output  
    - Outputs: Two code nodes extracting non-XML and XML links.  
    - Edge cases: Malformed XML or parsing errors.

  - **Sticky Note (Decompression)**  
    - Explains necessity to decompress `.gz` files and rename keys for unified processing.

#### 2.5 Link Extraction

- **Overview:** Extracts non-XML links and XML sitemap links from the parsed data.
- **Nodes Involved:**  
  - Extract non-xml links  
  - Extract xml links  
  - Sticky Notes (about sitemap links and binary checks)

- **Node Details:**  
  - **Extract non-xml links**  
    - Type: Code node  
    - Role: Extracts and outputs only the links that do not end with `.xml` or `.xml.gz`.  
    - Logic: Filters out sitemap.org and w3.org URLs, normalizes URLs, removes trailing punctuation, decodes HTML entities.  
    - Inputs: JSON parsed XML or text data  
    - Outputs: Append links to sheet node  
    - Edge cases: Unexpected input format, malformed URLs.

  - **Extract xml links**  
    - Type: Code node  
    - Role: Extracts URLs ending with `.xml` or `.xml.gz` for recursive scraping.  
    - Logic: Similar normalization and filtering as non-XML extraction but for XML URLs only.  
    - Inputs: JSON parsed XML or text data  
    - Outputs: Scrape xml file node (recursive)  
    - Edge cases: Cyclic recursion if sitemaps reference each other indefinitely (not explicitly handled).

  - **Sticky Note (If sitemap links are found)**  
    - Explains that if sitemap links exist, the workflow extracts XML links immediately.

  - **Sticky Note (Binary content check)**  
    - Explains that content may be received as text or binary, requiring conditional logic.

#### 2.6 Recursive Scraping and Storage

- **Overview:** Scrapes extracted XML links recursively and stores non-XML links in Google Sheets.
- **Nodes Involved:**  
  - Scrape xml file  
  - Append links to sheet  
  - Sticky Notes (about appending to sheet and recursive workflow)

- **Node Details:**  
  - **Scrape xml file**  
    - Type: ScrapingBee API node  
    - Role: Scrapes each extracted XML link for additional links.  
    - Config: Uses dynamic URL from extracted XML links.  
    - Credentials: ScrapingBee API credentials.  
    - Outputs: IF node to check binary (loop continues).  
    - Edge cases: Infinite recursion risk if sitemaps cyclically reference each other; API rate limits or failures.

  - **Append links to sheet**  
    - Type: Google Sheets node  
    - Role: Appends extracted non-XML links as rows in a Google Sheet under column "links".  
    - Config: Uses append mode, targets specific Google Sheet by URL and sheet ID (gid=0).  
    - Credentials: Google Sheets OAuth2 account.  
    - Inputs: Extract non-xml links output  
    - Edge cases: API authentication failure, quota limits, malformed data.

  - **Sticky Note (Add Links to Sheet)**  
    - Explains adding links to Google Sheet and recursive scraping of XML links.

  - **Sticky Note (Google Sheets setup)**  
    - Instructs to connect to Google Sheets and add "links" as a column.

#### 2.7 General Notes on Usage and Limitations

- **Sticky Note (Input)**  
  - Provides example URL for webhook input, emphasizing requirement of domain query parameter.

- **Sticky Note (Resource Usage Warning)**  
  - Warns that very large sitemaps may cause memory exhaustion and workflow crashes, suggesting upgrading plan or self-hosting.

---

### 3. Summary Table

| Node Name                      | Node Type                     | Functional Role                          | Input Node(s)              | Output Node(s)                     | Sticky Note                                                                                                                                                 |
|--------------------------------|-------------------------------|----------------------------------------|----------------------------|----------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Domain to scrape               | Webhook                       | Entry point, receives domain input     |                            | Scrape robots.txt file            | ## Input\nYou need to send a webhook request with domain as query parameter.\nFor example:\n`https://<webhook_link>?domain=n8n.io`                         |
| Scrape robots.txt file         | ScrapingBee API               | Scrapes robots.txt for sitemap links   | Domain to scrape            | If sitemap links are found, Scrape sitemap.xml file | ## Scrape Robots.txt\nMost websites provide sitemap links in robots.txt file so we will scrape it first                                                   |
| If sitemap links are found     | IF                           | Checks presence of sitemap links       | Scrape robots.txt file      | Extract xml links / Scrape sitemap.xml file | If sitemap links are are available, we will directly extract the xml links                                                                                  |
| Scrape sitemap.xml file        | ScrapingBee API               | Scrapes sitemap.xml if robots.txt lacks links | If sitemap links are found (false branch) | If it's a binary file              | ## Scrape Sitemap.xml\nIn case sitemap links are missing in robots.txt file, we will try to scrape sitemap.xml file                                        |
| If it's a binary file          | IF                           | Checks if response is binary            | Scrape sitemap.xml file, Scrape xml file | If it's a .gz file / Extract non-xml links, Extract xml links | Sometimes links are received as text content and sometimes they are received as binary, so we need to check for that.                                       |
| If it's a .gz file             | IF                           | Checks if binary file extension is .gz | If it's a binary file       | Decompress .gz file / Load the xml file as JSON | If it's a .xml.gz file, we need to decompress it. We are also renaming the key because by default they are named `file_0` and we need it to be named as `data` so that we can use a single extraction logic for both `.xml.gz` and `.xml` files |
| Decompress .gz file            | Compression                  | Decompresses .gz files                  | If it's a .gz file          | Store the file to data key for easy handling |                                                                                                                                                             |
| Store the file to data key for easy handling | Code                         | Renames binary key from file_0 to data | Decompress .gz file         | Load the xml file as JSON          |                                                                                                                                                             |
| Load the xml file as JSON      | Extract From File (XML)       | Converts XML binary to JSON             | Store the file to data key for easy handling | Extract non-xml links, Extract xml links |                                                                                                                                                             |
| Extract non-xml links          | Code                         | Extracts non-XML links from input       | Load the xml file as JSON, If it's a binary file | Append links to sheet             |                                                                                                                                                             |
| Extract xml links              | Code                         | Extracts XML sitemap links for recursion | Load the xml file as JSON, If it's a binary file, If sitemap links are found | Scrape xml file                   |                                                                                                                                                             |
| Scrape xml file               | ScrapingBee API               | Recursively scrapes extracted XML links | Extract xml links           | If it's a binary file              | ## Add Links to Sheet and Scrape XML Links\nIf the xml file contains normal links they are extracted and added to sheet. And if it contains other `.xml` links, we will scrape them. Basically, this is a recursive workflow. |
| Append links to sheet          | Google Sheets                 | Appends extracted links to Google Sheet | Extract non-xml links       |                                  | Connect to a Google Sheet and add `links` as column name                                                                                                   |
| Sticky Note                   | Sticky Note                  | Instructions and notes                  |                            |                                  | ## NOTE\nSome heavy sitemaps could result in a crash if the workflow consumes more memory than what is available in your n8n plan or self-hosted system. If this happens, we would recommend you to either upgrade your plan or use a self-hosted solution with a higher memory. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Name: Domain to scrape  
   - Type: Webhook  
   - Configure path to a unique value (e.g., `1da30868-fbca-4e8e-8580-485afb3fd956`)  
   - No authentication required  
   - Note: The webhook expects a query parameter named `domain`.

2. **Create ScrapingBee Node for robots.txt**  
   - Name: Scrape robots.txt file  
   - Type: ScrapingBee API  
   - URL: `https://{{ $json.query.domain }}/robots.txt` (use expression)  
   - Disable JavaScript rendering  
   - Credential: Link to your ScrapingBee account credentials  
   - Set retry on fail to true, continue on error as needed.

3. **Create IF Node to Check for Sitemap Links in robots.txt**  
   - Name: If sitemap links are found  
   - Condition: Check if `data` field (response body) contains string `Sitemap:`  
   - Use version 2 conditions with strict validation.

4. **Create ScrapingBee Node for sitemap.xml (fallback)**  
   - Name: Scrape sitemap.xml file  
   - Type: ScrapingBee API  
   - URL: `https://{{ $json.query.domain }}/sitemap.xml`  
   - Disable JS rendering  
   - Credential: ScrapingBee API  
   - Retry on fail enabled.

5. **Create IF Node to Check if Response is Binary**  
   - Name: If it's a binary file  
   - Condition: Check if `$binary` object is non-empty (object not empty)  
   - Used after scraping sitemap.xml and recursive XML scrapes.

6. **Create IF Node to Check if Binary File is .gz**  
   - Name: If it's a .gz file  
   - Condition: Check if `$binary.data.fileExtension` equals `gz`

7. **Create Compression Node to Decompress .gz Files**  
   - Name: Decompress .gz file  
   - No special parameters; set onError to continue regular output.

8. **Create Code Node to Rename Binary Key**  
   - Name: Store the file to data key for easy handling  
   - JavaScript: Move `file_0` binary property to `data` to standardize input for XML extraction.

9. **Create Extract From File Node to Parse XML**  
   - Name: Load the xml file as JSON  
   - Operation: XML extraction from `binary.data` property.

10. **Create Code Node to Extract Non-XML Links**  
    - Name: Extract non-xml links  
    - JavaScript logic to recursively extract all URLs excluding `.xml` or `.xml.gz` extensions and filtered domains.

11. **Create Code Node to Extract XML Links**  
    - Name: Extract xml links  
    - JavaScript logic to recursively extract all `.xml` and `.xml.gz` URLs.

12. **Create ScrapingBee Node for Recursive XML Scraping**  
    - Name: Scrape xml file  
    - URL set from extracted XML link (dynamic input)  
    - Credential: ScrapingBee API  
    - Retry enabled.

13. **Create Google Sheets Node to Append Links**  
    - Name: Append links to sheet  
    - Operation: Append rows  
    - Document ID: Set Google Sheet URL  
    - Sheet Name: Use sheet gid or name (`gid=0`)  
    - Mapping: Map `links` column to `$json.link`  
    - Credential: Google Sheets OAuth2 credentials.

14. **Connect nodes as per logic:**  
    - Domain to scrape → Scrape robots.txt file  
    - Scrape robots.txt file → If sitemap links are found  
    - If sitemap links are found (true) → Extract xml links  
    - If sitemap links are found (false) → Scrape sitemap.xml file  
    - Scrape sitemap.xml file → If it's a binary file  
    - If it's a binary file (true) → If it's a .gz file  
    - If it's a .gz file (true) → Decompress .gz file → Store file to data key → Load xml file as JSON  
    - If it's a .gz file (false) → Load xml file as JSON  
    - Load xml file as JSON → Extract non-xml links, Extract xml links  
    - Extract non-xml links → Append links to sheet  
    - Extract xml links → Scrape xml file (recursive)  
    - Scrape xml file → If it's a binary file (loop repeats)  
    - If it's a binary file (false) → Extract non-xml and xml links (continue recursion or terminate)  

15. **Add Sticky Notes with instructions:**  
    - Input usage instructions near webhook  
    - Notes near robots.txt scraping to explain rationale  
    - Notes on sitemap.xml fallback  
    - Notes on binary and `.gz` file handling  
    - Notes on appending to Google Sheets  
    - Warning about memory usage for large sitemaps.

---

### 5. General Notes & Resources

| Note Content                                                                                                         | Context or Link                                                                                                       |
|----------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------|
| Some heavy sitemaps could result in a crash if the workflow consumes more memory than available. Upgrade or self-host. | Workflow resource management advisory.                                                                                 |
| Input webhook requires `domain` query parameter, e.g., `https://<webhook_link>?domain=n8n.io`                       | Usage instructions for starting the workflow.                                                                         |
| Connect to a Google Sheet and add `links` as column name                                                            | Setup prerequisite for Google Sheets node.                                                                             |
| Recursive scraping handles `.xml` sitemaps linked from robots.txt or sitemap.xml                                     | Explains workflow recursion logic for sitemap discovery.                                                              |

---

**Disclaimer:** The provided text and workflow are generated exclusively from an n8n automation workflow and comply with content policies. All data handled are legal and public.