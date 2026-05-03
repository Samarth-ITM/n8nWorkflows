Build a Multi-Site Content Aggregator with Google Sheets & Custom Extraction Logic

https://n8nworkflows.xyz/workflows/build-a-multi-site-content-aggregator-with-google-sheets---custom-extraction-logic-11224


# Build a Multi-Site Content Aggregator with Google Sheets & Custom Extraction Logic

### 1. Workflow Overview

This workflow, titled **"Multi-Site Web Scraper with Source Routing"**, is designed to perform intelligent, automated content scraping from multiple distinct websites, each with custom extraction logic tailored to their unique HTML structures. It integrates tightly with Google Sheets to manage URLs to scrape and store the extracted article data.

The workflow is suitable for use cases such as aggregating news articles, blog posts, or other web content from multiple publishers, normalizing the extracted data, and maintaining freshness by filtering outdated content. It includes robust handling for varying source domains via a routing mechanism, rate limiting to respect target servers, and status tracking to monitor scraping success.

The workflow's logic is grouped into the following functional blocks:

- **1.1 Trigger & Input Reception:** Initiates the workflow manually or on a scheduled interval and reads URLs with associated source identifiers from a Google Sheet.
- **1.2 Rate Limiting & HTML Fetching:** Controls request pacing (3 seconds delay) and fetches the raw HTML content of each URL.
- **1.3 Source Routing:** Routes each fetched URL to a site-specific extraction logic block based on the source domain.
- **1.4 Site-Specific Content Extraction:** Extracts article metadata (title, description, author, publication date, image, canonical URL) using CSS selectors or fallback universal extraction.
- **1.5 Data Normalization:** Standardizes and cleans the extracted data fields.
- **1.6 Freshness Filtering:** Filters articles by publication date (default threshold: 45 days), marking older articles as outdated.
- **1.7 Tier & Status Calculation:** Assigns tier levels based on article age and timestamps extraction.
- **1.8 Output & Status Tracking:** Saves normalized and filtered data back to a Google Sheet and updates the status of each URL.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger & Input Reception

**Overview:**  
This block starts the workflow either manually or every 4 hours. It reads a list of URLs along with their source identifiers from a Google Sheets document.

**Nodes Involved:**  
- Manual Trigger  
- Schedule (Every 4 Hours)  
- Read Pending URLs  
- Loop Over URLs  
- Completion Summary  

**Node Details:**

- **Manual Trigger**  
  - Type: Trigger node (manual)  
  - Role: Allows manual start of the workflow.  
  - Inputs: None  
  - Outputs: Connects to "Read Pending URLs"  
  - Failure modes: None (manual trigger)  

- **Schedule (Every 4 Hours)**  
  - Type: Trigger node (time-based)  
  - Role: Automatically triggers the workflow every 4 hours.  
  - Configuration: Interval set to 4 hours  
  - Inputs: None  
  - Outputs: Connects to "Read Pending URLs"  
  - Failure modes: None (time-based trigger)  

- **Read Pending URLs**  
  - Type: Google Sheets node  
  - Role: Reads the sheet named "URLs to Process" from a configured Google Sheets document to retrieve URLs and their source identifiers.  
  - Configuration: Reads entire list from the specified sheet. Requires Google Sheets OAuth2 credentials.  
  - Expressions: None  
  - Inputs: Trigger nodes  
  - Outputs: Connects to "Loop Over URLs"  
  - Failure modes: Authentication errors, Google Sheets API limits, empty or malformed sheet data  

- **Loop Over URLs**  
  - Type: SplitInBatches  
  - Role: Processes URLs one at a time or in small batches to control flow and rate limiting.  
  - Configuration: Batch size defaults to 1 (not explicitly set), reset option disabled to continue batch processing.  
  - Inputs: "Read Pending URLs" output  
  - Outputs: Two branches: one to "Completion Summary" (summary per batch), another to "Rate Limit (3s)" (next processing steps)  
  - Failure modes: Batch processing errors; improper batch size may delay or overload later nodes  

- **Completion Summary**  
  - Type: Set node  
  - Role: Aggregates batch processing data, counting articles processed and timestamping completion.  
  - Configuration: Sets `articlesProcessed` as item count and `completedAt` as current ISO timestamp.  
  - Inputs: From "Loop Over URLs"  
  - Outputs: None (end branch)  
  - Failure modes: Expression evaluation errors  

---

#### 2.2 Rate Limiting & HTML Fetching

**Overview:**  
Ensures a 3-second delay between HTTP requests to avoid overloading target servers, then fetches the raw HTML content of each URL individually.

**Nodes Involved:**  
- Rate Limit (3s)  
- Fetch HTML  

**Node Details:**

- **Rate Limit (3s)**  
  - Type: Wait node  
  - Role: Delays execution for 3 seconds before continuing to the next node to respect rate limits.  
  - Configuration: Wait time of 3 seconds  
  - Inputs: From "Loop Over URLs"  
  - Outputs: Connects to "Fetch HTML"  
  - Failure modes: Timeout or workflow pause issues (unlikely)  

- **Fetch HTML**  
  - Type: HTTP Request node  
  - Role: Performs HTTP GET request to fetch the HTML content of the current URL.  
  - Configuration:  
    - URL: Expression `{{$json.URL}}` (dynamic URL from current item)  
    - Headers: User-Agent (standard browser), Accept headers to mimic browser request  
    - Timeout: 30 seconds  
    - Response format: Text (full HTML)  
  - Inputs: "Rate Limit (3s)"  
  - Outputs: Connects to "Source Router"  
  - Failure modes: Network errors, timeouts, 403/404/500 HTTP errors, anti-bot blocking, malformed URLs  
  - On error: Configured to continue workflow to avoid stopping on single URL failure  

---

#### 2.3 Source Routing

**Overview:**  
Routes each fetched HTML content to a site-specific extraction node based on the source identifier from the URL list. Has a fallback extraction for unknown sources.

**Nodes Involved:**  
- Source Router (Switch node)  

**Node Details:**

- **Source Router**  
  - Type: Switch node  
  - Role: Routes each item to one of five outputs based on the source domain string (Site A, Site B, Site C, Site D, fallback).  
  - Configuration:  
    - Condition: Checks if the `Source` field from "Loop Over URLs" equals one of the known site names.  
    - Case sensitive matching.  
    - Outputs: One output per site, plus a fallback output.  
  - Inputs: From "Fetch HTML"  
  - Outputs: Connects to respective site extractors or fallback extractor  
  - Failure modes: Missing or incorrect source values cause fallback to trigger; case sensitivity may cause misrouting  

---

#### 2.4 Site-Specific Content Extraction

**Overview:**  
Extracts structured article data using CSS selectors tailored to each site's HTML. Includes a fallback universal extractor implemented via code for unknown or new sources.

**Nodes Involved:**  
- Extract: Site A  
- Extract: Site B  
- Extract: Site C  
- Extract: Site D  
- Extract: Fallback (Universal)  

**Node Details:**

- **Extract: Site A / B / C / D**  
  - Type: HTML Extract node  
  - Role: Extracts fields like title, description, author, datePublished, imageUrl, canonicalUrl using CSS selectors and attribute extraction.  
  - Configuration:  
    - Site A example selectors:  
      - Title: `h1.article__title, h1[data-testid='ContentHeader'], .post-title h1`  
      - Description: meta tag with name='description' (attribute content)  
      - Author: selectors targeting author link or class names  
      - Date: datetime attribute of `time` tag  
      - Image URL: meta property 'og:image' content attribute  
      - Canonical URL: link rel='canonical' href attribute  
    - Sites B, C, D have similar but customized selectors matching their DOM structure.  
  - Inputs: From "Source Router" (corresponding output)  
  - Outputs: Connect to "Normalize Extracted Data"  
  - Failure modes: HTML structure changes break selectors; missing tags yield null fields; site blocking or incomplete HTML affects extraction  

- **Extract: Fallback (Universal)**  
  - Type: Code node (JavaScript)  
  - Role: Uses regex and heuristic parsing of raw HTML to extract the same fields as above in a generic way for unknown sites.  
  - Configuration:  
    - Extracts title from `<h1>`, `<title>`, or og:title meta  
    - Description from meta description, og:description, or first paragraphs  
    - Author from meta author, byline text, or JSON-LD structured data  
    - Date published from `<time>` tags, meta article:published_time, date patterns, or JSON-LD  
    - Image URL from og:image or twitter:image meta tags  
    - Canonical URL from link rel='canonical' or og:url meta  
  - Inputs: From "Source Router" fallback output  
  - Outputs: Connect to "Freshness Filter (45 days)"  
  - Failure modes: Regex might miss some patterns; JSON parsing errors in LD JSON if malformed; fallback may produce less accurate data  

---

#### 2.5 Data Normalization

**Overview:**  
Cleans and standardizes the extracted data fields to ensure uniformity, trimming whitespace, truncating descriptions, and filling missing canonical URLs.

**Nodes Involved:**  
- Normalize Extracted Data  

**Node Details:**

- **Normalize Extracted Data**  
  - Type: Code node  
  - Role: Processes each extracted item to trim spaces, limit description length to 1000 characters, and ensure canonical URL defaults to source URL if missing.  
  - Inputs: From all site extractors and fallback extractor  
  - Outputs: Connect to "Freshness Filter (45 days)"  
  - Failure modes: Expression or code errors (unlikely)  

---

#### 2.6 Freshness Filtering

**Overview:**  
Filters articles based on publication date, defaulting to a maximum age of 45 days. Outdated articles are separated and marked accordingly.

**Nodes Involved:**  
- Freshness Filter (45 days)  
- Mark as Outdated  

**Node Details:**

- **Freshness Filter (45 days)**  
  - Type: If node  
  - Role: Checks if `datePublished` is within the last 45 days.  
  - Configuration:  
    - Parses datePublished string, attempts ISO or cleaned formats  
    - Compares article date to current date minus 45 days  
    - If no date or invalid date, defaults to "fresh" (passes filter)  
  - Inputs: From "Normalize Extracted Data" and "Extract: Fallback (Universal)"  
  - Outputs:  
    - True branch: Connects to "Calculate Tier & Status"  
    - False branch: Connects to "Mark as Outdated"  
  - Failure modes: Date parsing errors; articles with missing dates pass filter unintentionally  

- **Mark as Outdated**  
  - Type: Set node  
  - Role: Marks articles as "Outdated" with a reason and retains source URL for tracking.  
  - Configuration:  
    - Sets `freshnessStatus` to "Outdated"  
    - Sets `reason` to "Article older than 45 days"  
    - Passes `sourceUrl` forward  
  - Inputs: From False branch of Freshness Filter  
  - Outputs: Connects to "Update URL Status"  
  - Failure modes: Data overwrite errors (unlikely)  

---

#### 2.7 Tier & Status Calculation

**Overview:**  
Assigns a tier label to each article based on its age, enriching metadata with extraction timestamp and freshness status.

**Nodes Involved:**  
- Calculate Tier & Status  

**Node Details:**

- **Calculate Tier & Status**  
  - Type: Code node  
  - Role:  
    - Parses `datePublished` and calculates days since publication  
    - Assigns tiers:  
      - ≤7 days: Tier 1  
      - ≤14 days: Tier 2  
      - ≤30 days: Tier 3  
      - >30 days: Archive  
    - Marks `freshnessStatus` as "Fresh" here (even for archive tier)  
    - Adds timestamp `extractedAt` as ISO string  
  - Inputs: From True branch of Freshness Filter  
  - Outputs: Connects to "Save to Article Feed"  
  - Failure modes: Date parsing errors, timezone issues  

---

#### 2.8 Output & Status Tracking

**Overview:**  
Saves cleaned and processed article data to the "Article Feed" sheet and updates the status of each processed URL in the "URLs to Process" sheet.

**Nodes Involved:**  
- Save to Article Feed  
- Update URL Status  

**Node Details:**

- **Save to Article Feed**  
  - Type: Google Sheets node  
  - Role: Appends or updates extracted article data in the "Article Feed" sheet.  
  - Configuration:  
    - Uses Google Sheets OAuth2 credentials  
    - Operation: appendOrUpdate  
    - Document and sheet names configured externally  
  - Inputs: From "Calculate Tier & Status"  
  - Outputs: Connects to "Update URL Status"  
  - Failure modes: API errors, quota limits, race conditions on updates  

- **Update URL Status**  
  - Type: Google Sheets node  
  - Role: Updates the status of each URL in "URLs to Process" sheet to reflect success or failure.  
  - Configuration:  
    - Operation: update  
    - Uses URL or sourceUrl as key for update  
    - Credentials as above  
  - Inputs: From "Save to Article Feed" and "Mark as Outdated"  
  - Outputs: Connects back to "Loop Over URLs" for continuous processing  
  - Failure modes: Sheet locking, concurrency issues, update conflicts  

---

### 3. Summary Table

| Node Name                     | Node Type             | Functional Role                 | Input Node(s)             | Output Node(s)           | Sticky Note                                                                                                   |
|-------------------------------|-----------------------|--------------------------------|---------------------------|--------------------------|---------------------------------------------------------------------------------------------------------------|
| Manual Trigger                | Trigger               | Manual workflow start          | None                      | Read Pending URLs         |                                                                                                               |
| Schedule (Every 4 Hours)      | Trigger               | Scheduled workflow start       | None                      | Read Pending URLs         |                                                                                                               |
| Read Pending URLs             | Google Sheets         | Reads URLs and sources         | Manual Trigger, Schedule  | Loop Over URLs            | Sticky Note - Input: "Reads URLs with source identifiers."                                                    |
| Loop Over URLs                | SplitInBatches        | Processes URLs one by one      | Read Pending URLs          | Completion Summary, Rate Limit (3s) |                                                                                                               |
| Completion Summary            | Set                   | Aggregates batch info          | Loop Over URLs             | None                     |                                                                                                               |
| Rate Limit (3s)               | Wait                  | Controls request pacing        | Loop Over URLs             | Fetch HTML                |                                                                                                               |
| Fetch HTML                   | HTTP Request          | Fetches raw HTML content       | Rate Limit (3s)            | Source Router             | Sticky Note - Input1: "Fetch the HTML content"                                                                |
| Source Router                | Switch                | Routes to site-specific extractors | Fetch HTML                | Extract: Site A/B/C/D, Extract: Fallback (Universal) | Sticky Note - Router: "Source Router (Switch)"                                                               |
| Extract: Site A              | HTML Extract          | Extracts Site A article data   | Source Router              | Normalize Extracted Data  | Sticky Note - Extractors: "Site-Specific Extractors\nCustom CSS selectors per publisher."                      |
| Extract: Site B              | HTML Extract          | Extracts Site B article data   | Source Router              | Normalize Extracted Data  | Sticky Note - Extractors                                                                                       |
| Extract: Site C              | HTML Extract          | Extracts Site C article data   | Source Router              | Normalize Extracted Data  | Sticky Note - Extractors                                                                                       |
| Extract: Site D              | HTML Extract          | Extracts Site D article data   | Source Router              | Normalize Extracted Data  | Sticky Note - Extractors                                                                                       |
| Extract: Fallback (Universal)| Code                  | Generic extraction fallback    | Source Router              | Freshness Filter (45 days)| Sticky Note - Extractors                                                                                       |
| Normalize Extracted Data     | Code                  | Cleans and standardizes data   | Extract: Site A/B/C/D, Extract: Fallback | Freshness Filter (45 days) |                                                                                                               |
| Freshness Filter (45 days)  | If                    | Filters articles by age        | Normalize Extracted Data, Extract: Fallback | Calculate Tier & Status, Mark as Outdated | Sticky Note - Freshness: "Filters articles by publication date."                                              |
| Calculate Tier & Status      | Code                  | Assigns tier and freshness     | Freshness Filter (45 days) | Save to Article Feed      | Sticky Note - Freshness1: "Tier Status\nCalculate Tier Status based on content"                                |
| Save to Article Feed         | Google Sheets         | Saves extracted article data   | Calculate Tier & Status    | Update URL Status         | Sticky Note - Output: "Output & Status Tracking\nSaves extracted data and updates source status."              |
| Mark as Outdated             | Set                   | Marks articles as outdated     | Freshness Filter (45 days) | Update URL Status         | Sticky Note - Output                                                                                           |
| Update URL Status            | Google Sheets         | Updates URL processing status  | Save to Article Feed, Mark as Outdated | Loop Over URLs            | Sticky Note - Output                                                                                           |
| Sticky Note - Introduction   | Sticky Note           | Workflow overview and instructions | None                      | None                     | Multi-Site Web Scraper with Source Routing explanation and setup steps                                         |
| Sticky Note - Input          | Sticky Note           | Describes input phase          | None                      | None                     | "Reads URLs with source identifiers."                                                                         |
| Sticky Note - Router         | Sticky Note           | Describes router node          | None                      | None                     | "Source Router (Switch)"                                                                                        |
| Sticky Note - Extractors     | Sticky Note           | Describes extractors           | None                      | None                     | "Site-Specific Extractors\nCustom CSS selectors per publisher."                                               |
| Sticky Note - Freshness      | Sticky Note           | Describes freshness filter     | None                      | None                     | "Filters articles by publication date."                                                                       |
| Sticky Note - Freshness1     | Sticky Note           | Describes tier status          | None                      | None                     | "Tier Status\nCalculate Tier Status based on content"                                                         |
| Sticky Note - Output         | Sticky Note           | Describes output phase         | None                      | None                     | "Output & Status Tracking\nSaves extracted data and updates source status."                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes:**
   - Add a **Manual Trigger** node to allow manual workflow starts.
   - Add a **Schedule Trigger** node configured to run every 4 hours.

2. **Create Google Sheets Node to Read URLs:**
   - Add a **Google Sheets** node named "Read Pending URLs".
   - Set operation to "Read" or "Get All".
   - Configure `documentId` to your Google Sheets document holding URLs.
   - Set `sheetName` to "URLs to Process".
   - Connect both trigger nodes to this node.
   - Configure Google Sheets OAuth2 credentials.

3. **Add SplitInBatches Node:**
   - Add a **SplitInBatches** node named "Loop Over URLs".
   - Connect "Read Pending URLs" to it.
   - Use default batch size 1 to process URLs sequentially.

4. **Add a Set Node for Completion Summary:**
   - Add a **Set** node named "Completion Summary".
   - Add fields:  
     - `articlesProcessed` = `{{$items().length}}` (number)  
     - `completedAt` = `{{$now.toISO()}}` (string)  
   - Connect a branch from "Loop Over URLs" to it.

5. **Add Wait Node for Rate Limiting:**
   - Add a **Wait** node named "Rate Limit (3s)".
   - Set wait time to 3 seconds.
   - Connect the other branch from "Loop Over URLs" to it.

6. **Add HTTP Request Node to Fetch HTML:**
   - Add an **HTTP Request** node named "Fetch HTML".
   - URL set to `{{$json.URL}}` from incoming data.
   - Set headers to mimic browser traffic (User-Agent, Accept, Accept-Language).
   - Set timeout to 30 seconds.
   - Configure to return full response as text.
   - Set "On Error" to continue workflow.
   - Connect "Rate Limit (3s)" to this node.

7. **Add Switch Node for Source Routing:**
   - Add a **Switch** node named "Source Router".
   - Configure outputs for each source domain: "Site A", "Site B", "Site C", "Site D", plus "fallback".
   - Use expression comparing `$('Loop Over URLs').item.json.Source` to each source string.
   - Connect "Fetch HTML" to this switch.

8. **Add HTML Extract Nodes for Each Site:**
   - For each source (Site A-D), add an **HTML Extract** node:
     - Configure CSS selectors for title, description, author, datePublished, imageUrl, canonicalUrl as per site structure.
     - Set operation to extract HTML content.
     - Set "On Error" to continue workflow.
     - Connect corresponding switch output to these nodes.

9. **Add Code Node for Universal Fallback Extraction:**
   - Add a **Code** node named "Extract: Fallback (Universal)".
   - Paste the provided JavaScript extraction code that uses regex and JSON-LD parsing.
   - Connect the fallback output of "Source Router" to this node.

10. **Add Code Node for Data Normalization:**
    - Add a **Code** node named "Normalize Extracted Data".
    - Paste JavaScript code that trims and normalizes extracted fields, truncates description to 1000 chars, and ensures canonical URL fallback.
    - Connect outputs of all extractors (Site A-D and fallback) to this node.

11. **Add If Node for Freshness Filtering:**
    - Add an **If** node named "Freshness Filter (45 days)".
    - Set condition to check if `datePublished` is within last 45 days.
    - Use custom JavaScript expression to parse date and compare.
    - Connect "Normalize Extracted Data" to this node.

12. **Add Code Node to Calculate Tier & Status:**
    - Add a **Code** node named "Calculate Tier & Status".
    - Paste JavaScript code that assigns tier based on article age and adds extraction timestamp.
    - Connect the True output branch of freshness filter to this node.

13. **Add Google Sheets Node to Save Article Feed:**
    - Add a **Google Sheets** node named "Save to Article Feed".
    - Set operation to "appendOrUpdate".
    - Configure document and sheet name as "Article Feed".
    - Connect "Calculate Tier & Status" to this node.
    - Use Google Sheets OAuth2 credentials.

14. **Add Set Node to Mark Outdated Articles:**
    - Add a **Set** node named "Mark as Outdated".
    - Assign `freshnessStatus` = "Outdated", `reason` = "Article older than 45 days", and pass `sourceUrl`.
    - Connect False output of "Freshness Filter (45 days)" to this node.

15. **Add Google Sheets Node to Update URL Status:**
    - Add a **Google Sheets** node named "Update URL Status".
    - Set operation to "update".
    - Configure document and sheet name as "URLs to Process".
    - Connect "Save to Article Feed" and "Mark as Outdated" nodes to this node.

16. **Close Loop:**
    - Connect "Update URL Status" output back to "Loop Over URLs" to continue processing next batch.

17. **Add Sticky Notes:**
    - Add sticky notes as per the workflow to document each functional block and key instructions.

---

### 5. General Notes & Resources

| Note Content                                                                                                               | Context or Link                                                                                          |
|----------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| Workflow intelligently routes URLs to site-specific extraction logic based on source domain.                               | See Sticky Note - Introduction, explains routing and extraction approach.                               |
| Rate limiting of 3 seconds is implemented to avoid server overload and potential IP blocking.                              | Sticky Note - Introduction and "Rate Limit (3s)" node configuration.                                    |
| Custom CSS selectors must be updated if source sites change their HTML structure.                                           | Sticky Note - Extractors; also applies to HTML Extract nodes.                                           |
| Freshness filter defaults to 45 days but can be adjusted in the If node called "Freshness Filter (45 days)".                | See sticky note describing freshness filtering logic.                                                  |
| For new sources, add new outputs to the Switch node and corresponding extractor nodes.                                     | Instructions provided in the main Sticky Note - Introduction block.                                     |
| Google Sheets document IDs and sheet names must be configured before running the workflow.                                 | Credentials and document references are placeholders requiring user setup.                              |
| On error, HTTP requests and extraction nodes continue workflow to avoid halting on single failures.                        | Configured via "On Error" setting in HTTP Request and HTML Extract nodes.                               |
| The fallback extractor uses regex and JSON-LD parsing as a generic method for unknown sites but may be less accurate.      | See "Extract: Fallback (Universal)" code node explanation.                                             |
| Article tiers assist downstream applications in prioritizing content freshness and relevance.                              | Tier calculation node "Calculate Tier & Status" adds metadata for further processing or filtering.      |

---

This documentation fully describes the workflow's structure, logic, and configuration, enabling reproduction, modification, and troubleshooting without needing the original JSON export.