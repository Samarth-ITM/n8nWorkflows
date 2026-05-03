Web Crawler: Convert Websites to AI-Ready Markdown in Google Sheets

https://n8nworkflows.xyz/workflows/web-crawler--convert-websites-to-ai-ready-markdown-in-google-sheets-9594


# Web Crawler: Convert Websites to AI-Ready Markdown in Google Sheets

### 1. Workflow Overview

This workflow automates crawling a website’s homepage to extract all sublinks, differentiate images from content pages, scrape textual content, convert it into AI-ready Markdown format, and aggregate all data into a Google Sheets document. It is designed for building structured knowledge bases, training AI datasets, competitor analysis, or dynamic site archiving.

The workflow is logically grouped into these blocks:

- **1.1 Trigger and Setup**: Manual or scheduled start and configuration of the target website URL.
- **1.2 Homepage Scraping and Link Extraction**: Fetch homepage HTML, extract all hyperlinks, and prepare them for processing.
- **1.3 Link Deduplication and Filtering**: Remove duplicates, filter only valid HTTPS URLs.
- **1.4 Link Categorization**: Separate image URLs from content page URLs using regex.
- **1.5 Content Scraping and Markdown Conversion**: Scrape content pages, convert HTML content into Markdown format.
- **1.6 Data Aggregation and Google Sheets Integration**: Aggregate images, links, and scraped content, then append or update them in Google Sheets.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger and Setup

**Overview:**  
This block initiates the workflow via manual trigger and sets the target website URL for crawling.

**Nodes Involved:**  
- Manual Trigger  
- Set Website  
- Note: Trigger and Setup

**Node Details:**

- **Manual Trigger**  
  - Type: Trigger node  
  - Role: Starts workflow execution manually  
  - Configuration: No parameters; simply triggers workflow  
  - Inputs: None  
  - Outputs: Connects to Set Website  
  - Edge cases: None typical; ensure manual start  
  - Version: 1

- **Set Website**  
  - Type: Set node  
  - Role: Assigns the target website URL into variable `website_url`  
  - Configuration: Static string field `website_url` set to `"https://example.com"` (to be updated)  
  - Inputs: From Manual Trigger  
  - Outputs: Connects to Scrape Homepage  
  - Edge cases: Ensure valid URL format to prevent HTTP request errors  
  - Version: 3.4

- **Note: Trigger and Setup**  
  - Type: Sticky Note  
  - Role: Documentation for user on how to configure trigger and website URL  
  - Configuration: Text content with instructions  
  - Inputs/Outputs: None

---

#### 1.2 Homepage Scraping and Link Extraction

**Overview:**  
Fetches the homepage HTML content and extracts all hyperlinks (`<a href>` tags), preparing a raw list of URLs for further processing.

**Nodes Involved:**  
- Scrape Homepage  
- Extract Links from HTML  
- Split Links  
- Note: Homepage Scraping

**Node Details:**

- **Scrape Homepage**  
  - Type: HTTP Request  
  - Role: Downloads homepage HTML from the `website_url` variable  
  - Configuration: URL dynamically set as `={{ $json.website_url }}`; redirects allowed; SSL cert validation enabled  
  - Inputs: From Set Website  
  - Outputs: Connects to Extract Links from HTML  
  - Edge cases: HTTP errors, redirects, invalid URLs, timeouts; configured to continue on error to avoid stopping workflow  
  - Version: 4.2

- **Extract Links from HTML**  
  - Type: HTML Extract  
  - Role: Parses HTML to extract all `href` attributes from `<a>` tags  
  - Configuration: Extracts array of href attributes under key `links`; trims and cleans text  
  - Inputs: From Scrape Homepage  
  - Outputs: Connects to Split Links  
  - Edge cases: Empty or malformed HTML, no links found  
  - Version: 1.2

- **Split Links**  
  - Type: Split Out  
  - Role: Splits the array of extracted links into individual items for processing  
  - Configuration: Field to split is `links`  
  - Inputs: From Extract Links from HTML  
  - Outputs: Connects to Remove Duplicate Links  
  - Edge cases: Empty arrays result in no output execution  
  - Version: 1

- **Note: Homepage Scraping**  
  - Type: Sticky Note  
  - Role: Documentation for homepage scraping process  
  - Inputs/Outputs: None

---

#### 1.3 Link Deduplication and Filtering

**Overview:**  
Removes duplicate URLs and filters links to keep only those starting with HTTPS, ensuring valid and unique hyperlinks proceed.

**Nodes Involved:**  
- Remove Duplicate Links  
- Filter Real Hyperlinks  
- Note: Link Processing

**Node Details:**

- **Remove Duplicate Links**  
  - Type: Remove Duplicates  
  - Role: Ensures uniqueness of links to avoid redundant processing  
  - Configuration: Default removal of duplicates from incoming list  
  - Inputs: From Split Links  
  - Outputs: Connects to Filter Real Hyperlinks  
  - Edge cases: Large number of duplicates may impact performance  
  - Version: 2

- **Filter Real Hyperlinks**  
  - Type: Filter  
  - Role: Filters links that start strictly with `"https://"`  
  - Configuration: Condition `startsWith` operator on `links` field; case sensitive  
  - Inputs: From Remove Duplicate Links  
  - Outputs: Connects to Separate Images and Links  
  - Edge cases: Non-HTTPS or malformed URLs filtered out; may exclude valid HTTP links if needed  
  - Version: 2.2

- **Note: Link Processing**  
  - Type: Sticky Note  
  - Role: Explains deduplication and link filtering logic  
  - Inputs/Outputs: None

---

#### 1.4 Link Categorization

**Overview:**  
Separates URLs into image links and content page links using regex matching on common image file extensions.

**Nodes Involved:**  
- Separate Images and Links  
- Aggregate Images  
- Aggregate Links  
- Note: Link Processing (continued)

**Node Details:**

- **Separate Images and Links**  
  - Type: Switch  
  - Role: Routes URLs into two outputs:
    - Output 0: Image URLs matching regex for extensions (png, jpg, jpeg, gif, webp, bmp, svg, ico)  
    - Output 1: Non-image URLs (content links)  
  - Configuration: Regex applied on the `links` field to identify images  
  - Inputs: From Filter Real Hyperlinks  
  - Outputs: 
    - Output 0 → Aggregate Images  
    - Output 1 → Aggregate Links and Scrape Content Links  
  - Edge cases: Regex might miss uncommon image formats or data URLs; adjust regex if needed  
  - Version: 3.2

- **Aggregate Images**  
  - Type: Aggregate  
  - Role: Collects all image URLs from output 0 into an array  
  - Configuration: Aggregates on `links` field  
  - Inputs: From Separate Images and Links (Images output)  
  - Outputs: Connects to Add Images to Sheet  
  - Edge cases: Empty image list leads to empty aggregation  
  - Version: 1

- **Aggregate Links**  
  - Type: Aggregate  
  - Role: Collects all non-image URLs into an array for further scraping  
  - Configuration: Aggregates on `links` field  
  - Inputs: From Separate Images and Links (Links output)  
  - Outputs: Connects to Add Links to Sheet  
  - Edge cases: Empty content link list means no content scraping  
  - Version: 1

- **Note: Link Processing**  
  - Sticky note already described above applies here as well.

---

#### 1.5 Content Scraping and Markdown Conversion

**Overview:**  
Scrapes the content pages from filtered links, converts the HTML content to Markdown format for AI readiness, then aggregates the Markdown content.

**Nodes Involved:**  
- Scrape Content Links  
- Convert to Markdown  
- Aggregate Scraped Content  
- Note: Content Scraping

**Node Details:**

- **Scrape Content Links**  
  - Type: HTTP Request  
  - Role: Downloads HTML content of each content page URL  
  - Configuration: URL dynamically set as `={{ $json.links }}` (individual link items)  
  - Inputs: From Aggregate Links (aggregated links are split before request)  
  - Outputs: Connects to Convert to Markdown  
  - Edge cases: HTTP errors, slow responses, invalid URLs; no explicit error handling, so failures may propagate  
  - Version: 4.2

- **Convert to Markdown**  
  - Type: Markdown  
  - Role: Converts scraped HTML content (`data` field) to Markdown format  
  - Configuration: Input HTML is `={{ $json.data }}`; default markdown options  
  - Inputs: From Scrape Content Links  
  - Outputs: Connects to Aggregate Scraped Content  
  - Edge cases: Complex HTML may convert imperfectly; large content sizes may be truncated later  
  - Version: 1

- **Aggregate Scraped Content**  
  - Type: Aggregate  
  - Role: Collects all Markdown content pieces into an array  
  - Configuration: Aggregates field `data` (Markdown content)  
  - Inputs: From Convert to Markdown  
  - Outputs: Connects to Add Scraped Content to Sheet  
  - Edge cases: Very large content arrays may reach Google Sheets cell limits  
  - Version: 1

- **Note: Content Scraping**  
  - Type: Sticky Note  
  - Purpose: Explains scraping, conversion, and aggregation; notes slice limits for Sheets  
  - Inputs/Outputs: None

---

#### 1.6 Data Aggregation and Google Sheets Integration

**Overview:**  
Writes aggregated images, links, and scraped Markdown content to a Google Sheets document, performing append or update based on the `Website` column.

**Nodes Involved:**  
- Add Images to Sheet  
- Add Links to Sheet  
- Add Scraped Content to Sheet  
- Note: Sheet Integration

**Node Details:**

- **Add Images to Sheet**  
  - Type: Google Sheets  
  - Role: Appends or updates the sheet with aggregated image URLs and website URL  
  - Configuration:
    - Columns mapped: Images as joined string of image URLs (`links.join('\n\n')`), Website as `website_url` from Set Website  
    - Matching column: `Website`  
    - Sheet name and Document ID placeholders (`your-sheet-name`, `your-document-id`) to be updated  
  - Inputs: From Aggregate Images  
  - Outputs: None  
  - Credentials: Google Sheets OAuth2 API (configured externally)  
  - Edge cases: Authorization errors, incorrect sheet name/ID, rate limits  
  - Version: 4.7

- **Add Links to Sheet**  
  - Type: Google Sheets  
  - Role: Appends or updates the sheet with aggregated content page links and website URL  
  - Configuration: Similar to Add Images, but column `Links`  
  - Inputs: From Aggregate Links  
  - Outputs: None  
  - Credentials: Same as above  
  - Edge cases: Same as above  
  - Version: 4.7

- **Add Scraped Content to Sheet**  
  - Type: Google Sheets  
  - Role: Appends or updates the sheet with aggregated Markdown content and website URL  
  - Configuration:
    - Columns: `Scraped Content` as concatenated Markdown strings sliced up to 50,000 characters to avoid cell size limits  
    - Matching column: `Website`  
    - Sheet name and Document ID placeholders  
  - Inputs: From Aggregate Scraped Content  
  - Outputs: None  
  - Credentials: Same as above  
  - Edge cases: Content truncation; sheet size and API limits; authorization errors  
  - Version: 4.7

- **Note: Sheet Integration**  
  - Type: Sticky Note  
  - Role: Explains how sheet integration works and necessity to update IDs and sheet names  
  - Inputs/Outputs: None

---

### 3. Summary Table

| Node Name                 | Node Type            | Functional Role                                   | Input Node(s)          | Output Node(s)                   | Sticky Note                                                                                                                                                                               |
|---------------------------|----------------------|-------------------------------------------------|------------------------|---------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Manual Trigger            | Trigger              | Starts workflow manually                          | None                   | Set Website                     | ## 🖱️ Trigger & Setup Nodes: Manual Trigger starts workflow; Set Website configures URL. Use Schedule Trigger for automation.                                                              |
| Set Website              | Set                  | Sets the target URL variable                      | Manual Trigger          | Scrape Homepage                 | Same as above.                                                                                                                                                                             |
| Scrape Homepage          | HTTP Request         | Downloads homepage HTML                           | Set Website             | Extract Links from HTML          | ## 🌐 Homepage Scraping Nodes: Fetch homepage HTML; extract links; handle redirects.                                                                                                       |
| Extract Links from HTML  | HTML Extract         | Extracts all href links from homepage HTML       | Scrape Homepage         | Split Links                    | Same as above.                                                                                                                                                                             |
| Split Links              | Split Out            | Splits link array to individual entries          | Extract Links from HTML | Remove Duplicate Links          | Same as above.                                                                                                                                                                             |
| Remove Duplicate Links   | Remove Duplicates    | Removes duplicate URLs                             | Split Links             | Filter Real Hyperlinks          | ## 🔄 Link Processing Nodes: Remove duplicates; filter HTTPS links; separate images and content links.                                                                                      |
| Filter Real Hyperlinks   | Filter               | Filters only HTTPS links                           | Remove Duplicate Links  | Separate Images and Links       | Same as above.                                                                                                                                                                             |
| Separate Images and Links| Switch               | Splits URLs into images and content links         | Filter Real Hyperlinks  | Aggregate Images, Aggregate Links, Scrape Content Links | Same as above.                                                                                                                                                                             |
| Aggregate Images         | Aggregate            | Aggregates all image URLs                          | Separate Images and Links (Images) | Add Images to Sheet             |                                                                                                                                                                                           |
| Add Images to Sheet      | Google Sheets        | Writes aggregated images to Google Sheets         | Aggregate Images        | None                          | ## 📊 Sheet Integration Nodes: Append/update images, links, content to Google Sheets; update documentId and sheetName.                                                                     |
| Aggregate Links          | Aggregate            | Aggregates all content page URLs                   | Separate Images and Links (Links) | Add Links to Sheet, Scrape Content Links |                                                                                                                                                                                           |
| Add Links to Sheet       | Google Sheets        | Writes aggregated links to Google Sheets           | Aggregate Links         | None                          | Same as above.                                                                                                                                                                             |
| Scrape Content Links     | HTTP Request         | Downloads HTML of content pages                     | Aggregate Links         | Convert to Markdown            | ## 📄 Content Scraping & Aggregation Nodes: Scrape content pages; convert to Markdown; aggregate content; slice if too large.                                                              |
| Convert to Markdown      | Markdown             | Converts HTML content to Markdown                   | Scrape Content Links    | Aggregate Scraped Content      | Same as above.                                                                                                                                                                             |
| Aggregate Scraped Content| Aggregate            | Aggregates all Markdown content                     | Convert to Markdown     | Add Scraped Content to Sheet   | Same as above.                                                                                                                                                                             |
| Add Scraped Content to Sheet | Google Sheets    | Writes aggregated Markdown content to Google Sheets| Aggregate Scraped Content | None                         | Same as above.                                                                                                                                                                             |
| Overview Note            | Sticky Note          | Workflow overview and instructions                  | None                   | None                          | # Automated Website Crawler for AI Knowledge Bases: Purpose, prerequisites, config steps, use cases, troubleshooting details.                                                              |
| Note: Trigger and Setup  | Sticky Note          | Instructions on triggering and setup                | None                   | None                          | See Manual Trigger and Set Website notes.                                                                                                                                                 |
| Note: Homepage Scraping  | Sticky Note          | Notes on homepage scraping nodes                     | None                   | None                          | See Scrape Homepage and Extract Links from HTML notes.                                                                                                                                     |
| Note: Link Processing    | Sticky Note          | Notes on deduplication, filtering, and categorization | None                   | None                          | See Remove Duplicate, Filter, Separate Images and Links notes.                                                                                                                             |
| Note: Content Scraping   | Sticky Note          | Notes on scraping content and converting to Markdown | None                   | None                          | See Scrape Content Links, Convert to Markdown, Aggregate Content notes.                                                                                                                    |
| Note: Sheet Integration  | Sticky Note          | Notes on Google Sheets integration, matching, and setup | None                   | None                          | See Add Images/Links/Scraped Content to Sheet notes.                                                                                                                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - No parameters needed  
   - Purpose: Start workflow manually

2. **Create Set Website Node**  
   - Type: Set  
   - Add field: `website_url` (string)  
   - Set value: `"https://example.com"` (replace with target site)  
   - Connect Manual Trigger → Set Website

3. **Create Scrape Homepage Node**  
   - Type: HTTP Request  
   - URL: `={{ $json.website_url }}`  
   - Options: Enable redirects, disable unauthorized certificates  
   - Connect Set Website → Scrape Homepage

4. **Create Extract Links from HTML Node**  
   - Type: HTML Extract  
   - Operation: Extract HTML content  
   - Extraction Values:  
     - key: `links`  
     - attribute: `href`  
     - cssSelector: `a`  
     - returnArray: true  
   - Trim and clean text enabled  
   - Connect Scrape Homepage → Extract Links from HTML

5. **Create Split Links Node**  
   - Type: Split Out  
   - Field to split: `links`  
   - Connect Extract Links from HTML → Split Links

6. **Create Remove Duplicate Links Node**  
   - Type: Remove Duplicates  
   - Default parameters  
   - Connect Split Links → Remove Duplicate Links

7. **Create Filter Real Hyperlinks Node**  
   - Type: Filter  
   - Condition: `links` startsWith `"https://"` (case sensitive)  
   - Connect Remove Duplicate Links → Filter Real Hyperlinks

8. **Create Separate Images and Links Node**  
   - Type: Switch  
   - Two outputs:  
     - Output 0 (Images): regex match `^https?:\/\/.*\.(png|jpe?g|gif|webp|bmp|svg|ico)(\?.*)?$` on `links`  
     - Output 1 (Links): not matching above regex  
   - Connect Filter Real Hyperlinks → Separate Images and Links

9. **Create Aggregate Images Node**  
   - Type: Aggregate  
   - Aggregate field: `links`  
   - Connect Separate Images and Links (Images output) → Aggregate Images

10. **Create Add Images to Sheet Node**  
    - Type: Google Sheets  
    - Operation: Append or Update  
    - Sheet Name: your sheet name  
    - Document ID: your document ID  
    - Columns mapping:  
      - Website: `={{ $('Set Website').item.json.website_url }}`  
      - Images: `={{ $json.links.join('\n\n') }}`  
    - Matching Columns: `Website`  
    - Set Google Sheets OAuth2 API credentials  
    - Connect Aggregate Images → Add Images to Sheet

11. **Create Aggregate Links Node**  
    - Type: Aggregate  
    - Aggregate field: `links`  
    - Connect Separate Images and Links (Links output) → Aggregate Links

12. **Create Add Links to Sheet Node**  
    - Same settings as Add Images but columns:  
      - Website: `={{ $('Set Website').item.json.website_url }}`  
      - Links: `={{ $json.links.join('\n\n') }}`  
    - Connect Aggregate Links → Add Links to Sheet

13. **Create Scrape Content Links Node**  
    - Type: HTTP Request  
    - URL: `={{ $json.links }}` (individual link items)  
    - Connect Aggregate Links → Scrape Content Links

14. **Create Convert to Markdown Node**  
    - Type: Markdown  
    - HTML Field: `={{ $json.data }}` (content from HTTP Request)  
    - Connect Scrape Content Links → Convert to Markdown

15. **Create Aggregate Scraped Content Node**  
    - Type: Aggregate  
    - Aggregate field: `data` (Markdown content)  
    - Connect Convert to Markdown → Aggregate Scraped Content

16. **Create Add Scraped Content to Sheet Node**  
    - Type: Google Sheets  
    - Operation: Append or Update  
    - Sheet Name: your sheet name  
    - Document ID: your document ID  
    - Columns mapping:  
      - Website: `={{ $('Set Website').item.json.website_url }}`  
      - Scraped Content: `={{ $json.data.join('\n\n').slice(0, 50000) }}` (limit to 50k chars)  
    - Matching Columns: `Website`  
    - Set Google Sheets OAuth2 API credentials  
    - Connect Aggregate Scraped Content → Add Scraped Content to Sheet

17. **Add Sticky Notes** (Optional for documentation)  
    - Add relevant sticky notes describing each block as per overview and analysis sections.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                      | Context or Link                                                                                                                       |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------|
| Google Sheets OAuth2 API Setup: create OAuth Client ID in Google Cloud Console; add n8n redirect URI; assign OAuth2 credentials in n8n with Sheets scopes.                                                                         | Instructions in Overview Note                                                                                                        |
| Use Schedule Trigger node to automate crawling instead of manual trigger.                                                                                                                                                        | Note: Trigger and Setup                                                                                                              |
| Adjust regex in Separate Images and Links node to capture additional image formats or different URL patterns.                                                                                                                   | Note: Link Processing                                                                                                                |
| Google Sheets cells have size limits; large content is sliced to 50,000 characters to avoid errors.                                                                                                                             | Note: Content Scraping                                                                                                               |
| If no links are extracted, verify homepage `<a>` tags and the accuracy of the target URL.                                                                                                                                       | Troubleshooting Overview Note                                                                                                       |
| Add Wait node after scraping HTTP requests to avoid rate limits or IP blocking.                                                                                                                                                   | Troubleshooting Overview Note                                                                                                       |
| Ensure Google Sheets document has columns: Website, Links, Scraped Content, Images.                                                                                                                                               | Overview Note                                                                                                                       |
| Workflow designed for n8n version supporting nodes with versions as listed (e.g., HTTP Request v4.2, Google Sheets v4.7).                                                                                                        | General compatibility note                                                                                                          |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and publicly accessible.