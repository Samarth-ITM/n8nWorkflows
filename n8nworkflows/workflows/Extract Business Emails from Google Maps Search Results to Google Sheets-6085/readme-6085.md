Extract Business Emails from Google Maps Search Results to Google Sheets

https://n8nworkflows.xyz/workflows/extract-business-emails-from-google-maps-search-results-to-google-sheets-6085


# Extract Business Emails from Google Maps Search Results to Google Sheets

### 1. Workflow Overview

This workflow automates the extraction of business email addresses from Google Maps search results based on a user-defined keyword. It is designed for users who want to gather targeted business contacts efficiently for outreach, marketing, recruitment, or research purposes.

The workflow is logically divided into four main blocks:

- **1.1 Input Reception & Initialization**  
  Accepts a keyword and triggers the entire process.

- **1.2 Collecting & Filtering Website URLs**  
  Queries Google Maps with the keyword, scrapes website URLs from the search results, and filters out invalid or duplicate URLs.

- **1.3 Email Extraction & Validation**  
  Iterates over each unique website URL to fetch content, extract potential emails, and validate them through regex matching and duplicate removal.

- **1.4 Aggregation & Storage**  
  Collects all verified emails, removes duplicates, and appends the results to a Google Sheet for easy access and follow-up.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Initialization

- **Overview:**  
  This block sets the target search keyword and triggers the workflow execution manually.

- **Nodes Involved:**  
  - When clicking ‘Test workflow’ (Manual Trigger)  
  - Fields - Set Keyword / Phrase (Set Node)

- **Node Details:**  

  1. **When clicking ‘Test workflow’**  
     - Type: Manual Trigger  
     - Role: Starts the workflow manually.  
     - Configuration: Default manual trigger with no parameters.  
     - Input: None  
     - Output: Triggers downstream nodes.  
     - Edge Cases: User must manually trigger; no automatic scheduling.  

  2. **Fields - Set Keyword / Phrase**  
     - Type: Set Node  
     - Role: Defines the keyword to search on Google Maps.  
     - Configuration: Sets a string field named `keyword` (default: "n8n workflow").  
     - Input: Trigger from Manual Trigger.  
     - Output: Passes keyword to next node.  
     - Edge Cases: Empty or invalid keyword may cause no results. User must update value manually.  

---

#### 1.2 Collecting & Filtering Website URLs

- **Overview:**  
  This block queries Google Maps with the keyword, extracts website URLs from the HTML response, filters out unwanted URLs, and removes duplicates to prepare a clean list of unique websites.

- **Nodes Involved:**  
  - HTTP Request - Get Sites  
  - Code - Matching URL  
  - Filter  
  - Remove Site Duplicates

- **Node Details:**  

  1. **HTTP Request - Get Sites**  
     - Type: HTTP Request  
     - Role: Sends a GET request to Google Maps search URL constructed with the keyword.  
     - Configuration: URL set as `https://www.google.com/maps/search/{{ $json.keyword }}`.  
     - Input: Keyword from Set node.  
     - Output: HTML/text response containing search results.  
     - Edge Cases: Possible errors include Google blocking automated requests (CAPTCHA), network timeouts, or no results.  

  2. **Code - Matching URL**  
     - Type: Code (JavaScript)  
     - Role: Parses the response HTML to extract website URLs using regex matching for URLs starting with http/https.  
     - Configuration: Uses regex to find `https?://[^/]+` patterns.  
     - Input: HTML data from previous HTTP request.  
     - Output: Array of JSON objects each containing a `url` property.  
     - Edge Cases: Malformed or changed Google Maps HTML structure may break regex matching.  

  3. **Filter**  
     - Type: Filter  
     - Role: Filters out URLs containing certain unwanted patterns such as “google”, “gstatic”, “ggpht”, “schema”, or “example”.  
     - Configuration: Condition is a negative regex match on the `url` field excluding those keywords.  
     - Input: URLs from Code node.  
     - Output: Only valid business website URLs pass through.  
     - Edge Cases: Overly aggressive filtering may remove valid URLs; insufficient filtering keeps irrelevant URLs.  

  4. **Remove Site Duplicates**  
     - Type: Remove Duplicates  
     - Role: Removes duplicate URLs to avoid repeated processing.  
     - Configuration: Compares based on `url` field.  
     - Input: Filtered URLs.  
     - Output: Unique list of website URLs.  
     - Edge Cases: URLs with minor differences (e.g., trailing slashes) may be treated as distinct.  

---

#### 1.3 Email Extraction & Validation

- **Overview:**  
  This block loops over each unique website URL to fetch website content, extract potential emails using regex, validate them, and collect all valid emails.

- **Nodes Involved:**  
  - Loop Websites (SplitInBatches)  
  - HTTP Request - Get Emails  
  - Loop Emails (SplitInBatches)  
  - Code - Match Email  
  - Aggregate

- **Node Details:**  

  1. **Loop Websites**  
     - Type: SplitInBatches  
     - Role: Iterates over each unique website URL one by one.  
     - Configuration: Default batch size and options.  
     - Input: Unique URLs from Remove Site Duplicates.  
     - Output: Single URL per batch for processing.  
     - Edge Cases: Large lists might slow processing; batch size adjustments possible.  

  2. **HTTP Request - Get Emails**  
     - Type: HTTP Request  
     - Role: Sends GET request to each website URL to fetch HTML content for email extraction.  
     - Configuration: URL set dynamically to current batch’s URL.  
     - Input: One URL per batch.  
     - Output: Website HTML/text content.  
     - Edge Cases: HTTP errors (404, 500), redirects, or blocked requests may occur.  

  3. **Loop Emails**  
     - Type: SplitInBatches  
     - Role: Loops over extracted emails to process one at a time.  
     - Configuration: Default batch size and options.  
     - Input: Emails array from Code - Match Email or HTTP Request? (Connected after Code - Match Email)  
     - Output: Single email per batch.  
     - Edge Cases: Empty email arrays or invalid formats handled downstream.  

  4. **Code - Match Email**  
     - Type: Code (JavaScript)  
     - Role: Extracts valid email addresses from website content using regex and excludes matches with blocked file extensions.  
     - Configuration: Uses regex pattern that avoids common file extensions (e.g., png, jpg, js). Returns matched email list.  
     - Input: Website HTML/text.  
     - Output: JSON with `emails` array.  
     - Edge Cases: Misses emails with unusual formats; false positives filtered by extension blocking.  

  5. **Aggregate**  
     - Type: Aggregate  
     - Role: Merges all batches of emails into a single list.  
     - Configuration: Merges on the `emails` field.  
     - Input: Emails from Loop Emails.  
     - Output: Combined emails array for deduplication.  
     - Edge Cases: Large email lists may impact performance.  

---

#### 1.4 Aggregation & Storage

- **Overview:**  
  This final block splits out emails, removes duplicates, and appends the unique emails along with the keyword to a Google Sheet.

- **Nodes Involved:**  
  - Split Out  
  - Remove Email Duplicates  
  - Google Sheets - Update Data

- **Node Details:**  

  1. **Split Out**  
     - Type: SplitOut  
     - Role: Splits the aggregated emails array into individual email entries.  
     - Configuration: Splits on the `emails` field.  
     - Input: Aggregated emails array.  
     - Output: Individual email entries for processing.  
     - Edge Cases: Empty arrays output nothing; no special failures expected.  

  2. **Remove Email Duplicates**  
     - Type: Remove Duplicates  
     - Role: Removes duplicate emails to ensure uniqueness.  
     - Configuration: Compares on `emails` field.  
     - Input: Split emails.  
     - Output: Unique email entries.  
     - Edge Cases: Case sensitivity may affect duplicates; normalized casing recommended if needed.  

  3. **Google Sheets - Update Data**  
     - Type: Google Sheets  
     - Role: Appends unique emails with the keyword to a specified Google Sheet.  
     - Configuration:  
       - Operation: Append  
       - Sheet Name: `gid=0` (default first sheet)  
       - Document ID: `17y_MmRHfBbW67bVyRuep1hkpoh2BV3yF5_Woxt0W9kk`  
       - Columns: `Emails` and `Keyword` (keyword taken from the Set node)  
       - Credential: Google Sheets OAuth2 credential named "Google Sheets - toan.ngo"  
     - Input: Unique emails and keyword.  
     - Output: Confirmation of append operation.  
     - Edge Cases: Credential expiry, API quota limits, or sheet permission errors may occur.  

---

### 3. Summary Table

| Node Name                       | Node Type          | Functional Role                                | Input Node(s)                   | Output Node(s)                  | Sticky Note                                                                                              |
|--------------------------------|--------------------|-----------------------------------------------|--------------------------------|--------------------------------|--------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’   | Manual Trigger     | Start workflow manually                        | None                           | Fields - Set Keyword / Phrase   | ## 1. Set The Keyword & Start The Workflow - A target keyword should be entered in Fields - Set Keyword / Phrase. |
| Fields - Set Keyword / Phrase   | Set                | Define search keyword                          | When clicking ‘Test workflow’  | HTTP Request - Get Sites        | ## Change Keyword Here                                                                                   |
| HTTP Request - Get Sites        | HTTP Request       | Query Google Maps with keyword                 | Fields - Set Keyword / Phrase  | Code - Matching URL             | ## 2. Collect & Filter Website URLs - Queries Google Maps and collects URLs from HTML.                   |
| Code - Matching URL             | Code               | Extract URLs from Google Maps HTML             | HTTP Request - Get Sites       | Filter                         |                                                                                                        |
| Filter                         | Filter             | Remove invalid URLs                             | Code - Matching URL            | Remove Site Duplicates          |                                                                                                        |
| Remove Site Duplicates          | Remove Duplicates  | Remove duplicate website URLs                   | Filter                        | Loop Websites                  |                                                                                                        |
| Loop Websites                  | SplitInBatches     | Loop over each unique website URL               | Remove Site Duplicates         | HTTP Request - Get Emails, Loop Emails | ## 3. Extract & Validate Emails - Loop over URLs to extract emails and validate.                       |
| HTTP Request - Get Emails       | HTTP Request       | Fetch website content for email extraction     | Loop Websites                 | Loop Websites                  |                                                                                                        |
| Loop Emails                   | SplitInBatches     | Loop over each extracted email                   | Loop Websites, Code - Match Email | Aggregate, Code - Match Email |                                                                                                        |
| Code - Match Email             | Code               | Validate and extract properly formatted emails | Loop Emails                   | Loop Emails                   |                                                                                                        |
| Aggregate                      | Aggregate          | Merge all emails into a single list              | Loop Emails                   | Split Out                     | ## 4. Aggregate, Deduplicate & Save Results - Collects and deduplicates emails before saving.           |
| Split Out                     | SplitOut           | Split aggregated emails into individual entries | Aggregate                    | Remove Email Duplicates        |                                                                                                        |
| Remove Email Duplicates         | Remove Duplicates  | Remove duplicate emails                          | Split Out                    | Google Sheets - Update Data    |                                                                                                        |
| Google Sheets - Update Data     | Google Sheets      | Append unique emails and keyword to Google Sheet| Remove Email Duplicates       | None                         |                                                                                                        |
| Sticky Note                    | Sticky Note        | Instructional note                              | None                         | None                         | ## 1. Set The Keyword & Start The Workflow - Explains keyword input and trigger.                        |
| Sticky Note1                   | Sticky Note        | Instructional note                              | None                         | None                         | ## Change Keyword Here                                                                                   |
| Sticky Note2                   | Sticky Note        | Instructional note                              | None                         | None                         | ## 2. Collect & Filter Website URLs - Explains URL scraping and filtering.                               |
| Sticky Note3                   | Sticky Note        | Instructional note                              | None                         | None                         | ## [n8n Automation] Automated Email Extractor By Keyword From Google Maps - Full workflow overview and setup instructions. |
| Sticky Note4                   | Sticky Note        | Instructional note                              | None                         | None                         | ## 3. Extract & Validate Emails - Describes email extraction and validation process.                    |
| Sticky Note5                   | Sticky Note        | Instructional note                              | None                         | None                         | ## 4. Aggregate, Deduplicate & Save Results - Describes final aggregation and saving to Google Sheets.   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Purpose: To start the workflow manually.

2. **Create Set Node: Fields - Set Keyword / Phrase**  
   - Assign a string field named `keyword`.  
   - Default value: e.g., "n8n workflow".  
   - Connect Manual Trigger output to this node.

3. **Create HTTP Request Node: HTTP Request - Get Sites**  
   - Method: GET  
   - URL: `https://www.google.com/maps/search/{{ $json.keyword }}` (use expression editor to insert keyword)  
   - Connect from Set node.

4. **Create Code Node: Code - Matching URL**  
   - Language: JavaScript  
   - Code snippet:  
     ```javascript
     const urls = $input.first().json.data.match(/https?:\/\/[^\/]+/g);
     return urls.map(url => ({json: {url: url}}));
     ```  
   - Connect from HTTP Request - Get Sites.

5. **Create Filter Node: Filter**  
   - Condition: `url` does NOT match regex `(google|gstatic|ggpht|schema|example)`  
   - Connect from Code - Matching URL.

6. **Create Remove Duplicates Node: Remove Site Duplicates**  
   - Compare field: `url`  
   - Connect from Filter.

7. **Create SplitInBatches Node: Loop Websites**  
   - Use default batch size (or adjust for performance).  
   - Connect from Remove Site Duplicates.

8. **Create HTTP Request Node: HTTP Request - Get Emails**  
   - Method: GET  
   - URL: `={{ $json.url }}` (dynamic URL from batch)  
   - Connect from Loop Websites.

9. **Connect HTTP Request - Get Emails output to Loop Websites node (main output 1)**  
   - This creates a loop back to Loop Websites for batch continuation.

10. **Create SplitInBatches Node: Loop Emails**  
    - Default batch size.  
    - Connect from Loop Websites (main output 0).

11. **Create Code Node: Code - Match Email**  
    - JavaScript code:  
      ```javascript
      const blockExtensions = [
        'png', 'jpg', 'jpeg', 'gif', 'webp', 'svg',
        'mp4', 'avi', 'mov', 'webm', 'mkv',
        'js', 'css', 'json', 'ts', 'jsx', 'tsx'
      ];

      const regex = new RegExp(
        `[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.(?!${blockExtensions.join('|')})[a-zA-Z]{2,}`,
        'gi'
      );

      const text = $input.first().json.data;
      const emails = text.match(regex);

      return { json: { emails: emails } };
      ```  
    - Connect main output 1 of Loop Emails to this node.

12. **Connect Code - Match Email output back to Loop Emails node (main output 0)**  
    - To continue looping.

13. **Create Aggregate Node: Aggregate**  
    - Operation: Merge lists on field `emails`.  
    - Connect from Loop Emails (main output 0).

14. **Create SplitOut Node: Split Out**  
    - Field to split out: `emails`  
    - Connect from Aggregate.

15. **Create Remove Duplicates Node: Remove Email Duplicates**  
    - Compare field: `emails`  
    - Connect from Split Out.

16. **Create Google Sheets Node: Google Sheets - Update Data**  
    - Operation: Append  
    - Document ID: `17y_MmRHfBbW67bVyRuep1hkpoh2BV3yF5_Woxt0W9kk` (replace with your sheet ID)  
    - Sheet Name: `gid=0` or your desired sheet  
    - Columns: Map `Emails` to `{{$json.emails}}`, `Keyword` to `{{$node["Fields - Set Keyword / Phrase"].json.keyword}}`  
    - Credential: Set up Google Sheets OAuth2 credentials with proper scopes for sheet access.  
    - Connect from Remove Email Duplicates.

17. **Set up connections as per workflow:**  
    - Manual Trigger → Set Keyword → HTTP Request - Get Sites → Code - Matching URL → Filter → Remove Site Duplicates → Loop Websites  
    - Loop Websites → HTTP Request - Get Emails → Loop Websites (for batch continuation)  
    - Loop Websites → Loop Emails → Code - Match Email → Loop Emails (batch continuation) → Aggregate → Split Out → Remove Email Duplicates → Google Sheets - Update Data

18. **Credential setup:**  
    - Google Sheets OAuth2 credential: enables appending data to Google Sheets.  
    - No special credentials needed for HTTP requests unless Google Maps blocks requests (then proxy or API needed).  

19. **Test and run the workflow manually.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | Context or Link                                                                                      |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| This workflow automates the extraction of business emails from Google Maps search results by keyword, ideal for sales, marketing, recruiting, and research teams. It eliminates manual email collection and integrates results into Google Sheets.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       | Sticky Note content in workflow, overview of use cases and target audience.                        |
| To use, duplicate the provided Google Sheets template and set up Google Cloud OAuth credentials with Sheets API enabled.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    | Google Sheets template link: https://docs.google.com/spreadsheets/d/17y_MmRHfBbW67bVyRuep1hkpoh2BV3yF5_Woxt0W9kk/edit#gid=0 |
| Google Maps may block automated scraping; consider legal and technical limitations. Use with caution and respect Google’s terms.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              | General caution regarding scraping Google Maps.                                                   |
| For customization, adjust the keyword in the Set node or add follow-up actions like email sending.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       | Customization notes from Sticky Notes.                                                           |
| Support and community resources available via Agent Circle channels (Discord, YouTube, LinkedIn, etc.)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   | https://www.agentcircle.ai/ and associated social links.                                         |

---

**Disclaimer:** The provided content is exclusively from an automated workflow created with n8n, adhering strictly to content policies and containing no illegal or protected data. All data processed is publicly accessible and lawful.