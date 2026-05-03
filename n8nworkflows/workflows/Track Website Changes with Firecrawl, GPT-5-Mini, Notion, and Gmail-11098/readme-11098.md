Track Website Changes with Firecrawl, GPT-5-Mini, Notion, and Gmail

https://n8nworkflows.xyz/workflows/track-website-changes-with-firecrawl--gpt-5-mini--notion--and-gmail-11098


# Track Website Changes with Firecrawl, GPT-5-Mini, Notion, and Gmail

---

# Track Website Changes with Firecrawl, GPT-5-Mini, Notion, and Gmail

---

## 1. Workflow Overview

This n8n workflow automates the monitoring of specified websites for meaningful content changes, capturing snapshots, analyzing differences using AI, storing results in Notion, and optionally sending email alerts. It is designed for users who wish to track updates on multiple websites efficiently without manual checks.

The workflow is logically divided into these blocks:

- **1.1 Scheduled Trigger & Input Setup:** Defines when the workflow runs and the list of target URLs to monitor, along with the criteria defining meaningful changes.
- **1.2 Website Data Capture:** Scrapes current website content and screenshots using Firecrawl for each target URL.
- **1.3 Snapshot Storage in Notion:** Saves website snapshots (content and screenshots) to a Notion database, splitting large content into manageable blocks.
- **1.4 Historical Snapshot Retrieval:** Retrieves the last saved snapshot for each website from Notion for comparison.
- **1.5 AI-Powered Comparison:** Uses GPT-5-Mini to compare current and previous snapshots, summarizing detected meaningful changes.
- **1.6 Update Creation and Notification:** Creates update entries with detailed comparisons in Notion and optionally sends formatted email alerts via Gmail.

---

## 2. Block-by-Block Analysis

### 1.1 Scheduled Trigger & Input Setup

**Overview:**  
Initial block to trigger the workflow on a schedule and define the target URLs along with change criteria.

**Nodes Involved:**  
- Schedule Trigger  
- Define Target URLs  
- Split Out  
- Define What Matters To You

**Node Details:**

- **Schedule Trigger**  
  - Type: Trigger  
  - Role: Starts workflow execution on a recurring schedule (daily by default).  
  - Configuration: Interval set to daily (default empty interval object).  
  - Inputs: None  
  - Outputs: Connects to "Define Target URLs".  
  - Edge Cases: If trigger fails, workflow will not start; ensure n8n scheduler is active.

- **Define Target URLs**  
  - Type: Set  
  - Role: Defines a JSON array of URLs to monitor.  
  - Configuration: Hardcoded JSON array with example URLs ["https://randomtextgenerator.com", "https://randomuser.me/"].  
  - Inputs: From Schedule Trigger  
  - Outputs: Connects to "Split Out".  
  - Edge Cases: Empty or malformed URLs will cause downstream nodes to fail.

- **Split Out**  
  - Type: SplitOut  
  - Role: Splits the array of URLs into individual items to process each URL separately.  
  - Configuration: Field to split out: "target_urls".  
  - Inputs: From "Define Target URLs"  
  - Outputs: Connects to "Define What Matters To You".  
  - Edge Cases: If input array is empty, no further processing occurs.

- **Define What Matters To You**  
  - Type: Set  
  - Role: Defines the textual criteria that specify what counts as a meaningful change.  
  - Configuration: String field with value: "Either a visible text change or an image source update should be considered a change".  
  - Inputs: From "Split Out"  
  - Outputs: Connects to "Scrape Target URLs" (via connection from "Define What Matters To You").  
  - Edge Cases: Criteria should be clear and concise to guide AI analysis; ambiguous criteria could reduce accuracy.

---

### 1.2 Website Data Capture

**Overview:**  
Scrapes the current website content and captures screenshots using Firecrawl for each URL.

**Nodes Involved:**  
- Scrape Target URLs  
- Capture Screenshots

**Node Details:**

- **Scrape Target URLs**  
  - Type: Firecrawl (scrape operation)  
  - Role: Scrapes website content (including text and metadata) for each URL.  
  - Configuration:  
    - URL set dynamically from current split item.  
    - Wait 5000ms before scraping to allow page load.  
    - OnlyMainContent: false (scrapes full page).  
    - Formats: default (empty format object implies default content scraping).  
  - Credentials: Firecrawl API key required.  
  - Inputs: From "Define What Matters To You"  
  - Outputs: Connects to "Capture Screenshots" and "Get Last Snapshots".  
  - Edge Cases:  
    - Network failures or invalid URLs cause scrape failure.  
    - Page load delays longer than wait time may cause incomplete scrape.  
    - Firecrawl API quota limits or authentication errors.

- **Capture Screenshots**  
  - Type: Firecrawl (scrape operation)  
  - Role: Captures full-page screenshots of each target URL.  
  - Configuration:  
    - URL from current item.  
    - Wait 5000ms.  
    - Format: screenshot with fullPage=true.  
  - Credentials: Firecrawl API key.  
  - Inputs: From "Scrape Target URLs"  
  - Outputs: Connects to "Save Snapshots" and "Get Last Snapshots".  
  - Edge Cases: Similar to scrape node; image capture failures possible due to page complexity or API limits.

---

### 1.3 Snapshot Storage in Notion

**Overview:**  
Stores the scraped content and screenshots as snapshots in a Notion database, splitting content into manageable blocks.

**Nodes Involved:**  
- Save Snapshots  
- Construct Notion Blocks  
- Split Out Blocks  
- Append Blocks

**Node Details:**

- **Save Snapshots**  
  - Type: Notion (databasePage resource)  
  - Role: Creates a new page in the "Snapshots" database for each URL snapshot.  
  - Configuration:  
    - Database ID set to Notion "Snapshots" database.  
    - Properties set: Name with date and URL, screenshot file URL, URL property, created date.  
  - Credentials: Notion API key.  
  - Inputs: From "Capture Screenshots"  
  - Outputs: Connects to "Construct Notion Blocks".  
  - Edge Cases:  
    - Notion API limits or permission errors.  
    - Failed upload of screenshot URL may result in missing images.

- **Construct Notion Blocks**  
  - Type: Code  
  - Role: Processes all saved snapshot pages and their scraped markdown content to split into blocks of max 1500 characters (below Notion block limit).  
  - Configuration: JavaScript iterates over pages and scraped results; slices content into chunks, prepares objects with pageId and chunk text.  
  - Inputs: Paired input from "Save Snapshots" and "Scrape Target URLs".  
  - Outputs: Returns array of blocks for appending.  
  - Edge Cases:  
    - Mismatch in count of pages and scraped results throws error.  
    - Large content handled by chunking; very large pages could increase execution time.

- **Split Out Blocks**  
  - Type: SplitOut  
  - Role: Splits the array of blocks into individual block items for appending.  
  - Configuration: Field to split out: "blocks".  
  - Inputs: From "Construct Notion Blocks"  
  - Outputs: Connects to "Append Blocks".  
  - Edge Cases: Empty blocks array stops further processing.

- **Append Blocks**  
  - Type: Notion (block resource)  
  - Role: Appends text blocks to the corresponding Notion page created earlier.  
  - Configuration:  
    - Block ID dynamically set from chunk's pageId.  
    - Text content set to chunk content.  
  - Credentials: Notion API key.  
  - Inputs: From "Split Out Blocks"  
  - Outputs: None (end of this branch).  
  - Edge Cases:  
    - Notion API failures or rate limiting.  
    - Incorrect pageId leads to append failure.

---

### 1.4 Historical Snapshot Retrieval

**Overview:**  
Retrieves the last snapshot for each target URL from the Notion "Snapshots" database, including all child blocks, for comparison.

**Nodes Involved:**  
- Get Last Snapshots  
- Get Many Blocks

**Node Details:**

- **Get Last Snapshots**  
  - Type: Notion (databasePage resource)  
  - Role: Fetches the most recent snapshot for each URL created on or before yesterday.  
  - Configuration:  
    - Database ID for Snapshots.  
    - Filters: URL equals current target URL, created date on or before yesterday.  
    - Sort descending by created date.  
    - Limit: 1 (only latest snapshot).  
  - Credentials: Notion API key.  
  - Inputs: From "Capture Screenshots"  
  - Outputs: Connects to "Get Many Blocks".  
  - Edge Cases:  
    - No previous snapshot found means first run or missing data.  
    - Notion API errors or filter misconfiguration.

- **Get Many Blocks**  
  - Type: Notion (block resource)  
  - Role: Retrieves all blocks (paragraphs, text) under the last snapshot page.  
  - Configuration:  
    - Block ID dynamically set from last snapshot page ID.  
    - Returns all child blocks.  
  - Credentials: Notion API key.  
  - Inputs: From "Get Last Snapshots"  
  - Outputs: Connects to "Combine Website Data".  
  - Edge Cases:  
    - Large pages with many blocks may slow response.  
    - Missing or deleted blocks.

---

### 1.5 AI-Powered Comparison

**Overview:**  
Combines current and previous snapshot data, then uses GPT-5-Mini to compare content based on defined criteria and produces concise summaries of meaningful changes.

**Nodes Involved:**  
- Combine Website Data  
- Split Out Data  
- Compare Snapshots  
- Combine Snapshots and AI Analysis  
- Split Out Updates

**Node Details:**

- **Combine Website Data**  
  - Type: Code  
  - Role: Aggregates data for each target URL: current content and screenshot, last snapshot content and screenshot, last snapshot date.  
  - Configuration: JavaScript merges data from multiple previous nodes, filtering and matching by URL.  
  - Inputs: From "Get Many Blocks", "Get Last Snapshots", "Scrape Target URLs", "Capture Screenshots"  
  - Outputs: Produces array "websites" with combined info per URL.  
  - Edge Cases:  
    - Missing last snapshot skips URL.  
    - Length mismatch in input arrays throws error.

- **Split Out Data**  
  - Type: SplitOut  
  - Role: Splits combined website data array into individual items for AI processing.  
  - Configuration: Field to split out: "websites".  
  - Inputs: From "Combine Website Data"  
  - Outputs: Connects to "Compare Snapshots".  
  - Edge Cases: Empty array results in no AI calls.

- **Compare Snapshots**  
  - Type: OpenAI (via LangChain node)  
  - Role: Sends prompt to GPT-5-Mini model to compare current and previous content and return meaningful change summary or "NO_CHANGE".  
  - Configuration:  
    - Model: gpt-5-mini.  
    - Prompt includes URL, dates, change criteria, current and previous content.  
  - Credentials: OpenAI API key.  
  - Inputs: From "Split Out Data"  
  - Outputs: Connects to "Combine Snapshots and AI Analysis".  
  - Edge Cases:  
    - API rate limits, timeouts, or quota exceeded.  
    - Ambiguous prompt or very large content may cause incomplete or irrelevant responses.

- **Combine Snapshots and AI Analysis**  
  - Type: Code  
  - Role: Filters AI analysis results to keep only meaningful changes (excludes "NO_CHANGE"), compiling updates for further processing.  
  - Configuration: JavaScript matches AI responses with website data; skips items with "NO_CHANGE".  
  - Inputs: From "Compare Snapshots" and "Split Out Data"  
  - Outputs: Array "updates" with summaries and details.  
  - Edge Cases: Length mismatch throws error.

- **Split Out Updates**  
  - Type: SplitOut  
  - Role: Splits updates into individual items for processing.  
  - Configuration: Field to split out: "updates".  
  - Inputs: From "Combine Snapshots and AI Analysis"  
  - Outputs: Connects to "Create Updates" and "Send Email Updates".  
  - Edge Cases: Empty updates array means no further steps executed.

---

### 1.6 Update Creation and Notification

**Overview:**  
Creates update pages in Notion with detailed comparisons and optionally sends email alerts summarizing detected changes.

**Nodes Involved:**  
- Create Updates  
- Construct Update Pages  
- Send Email Updates

**Node Details:**

- **Create Updates**  
  - Type: Notion (databasePage resource)  
  - Role: Creates a new page in the "Updates" database for each detected update.  
  - Configuration:  
    - Database ID for Updates.  
    - Properties: Name with date and URL, created date, URL property.  
  - Credentials: Notion API key.  
  - Inputs: From "Split Out Updates"  
  - Outputs: Connects to "Construct Update Pages".  
  - Edge Cases: Notion API errors or permission issues.

- **Construct Update Pages**  
  - Type: HTTP Request (Notion API direct call)  
  - Role: Adds rich content blocks (columns with current and previous screenshots side by side and summary text) to the created update page.  
  - Configuration:  
    - PATCH request to Notion API blocks children endpoint for the created page ID.  
    - JSON body creates a column list with two columns: current snapshot and previous snapshot headings and images, plus a summary paragraph.  
    - Uses dynamic expressions to insert dates, URLs, screenshots, and summary.  
  - Credentials: Notion API key.  
  - Inputs: From "Create Updates"  
  - Outputs: None (end branch).  
  - Edge Cases:  
    - Notion API rate limits, malformed JSON, or invalid page ID cause failure.

- **Send Email Updates**  
  - Type: Gmail (OAuth2)  
  - Role: Sends a styled HTML email notification with current and previous screenshots and summary for each detected update.  
  - Configuration:  
    - Recipient email hardcoded as "your-mail@example.com" (should be updated).  
    - Subject dynamically includes the URL domain.  
    - HTML body includes formatted content with images and summary.  
  - Credentials: Gmail OAuth2 credentials.  
  - Inputs: From "Split Out Updates"  
  - Outputs: None (end branch).  
  - Edge Cases:  
    - Email sending failures due to auth errors or Gmail quota.  
    - Recipient email must be valid.

---

## 3. Summary Table

| Node Name               | Node Type                | Functional Role                                       | Input Node(s)                  | Output Node(s)                             | Sticky Note                                                                                                                           |
|-------------------------|--------------------------|------------------------------------------------------|--------------------------------|--------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger        | scheduleTrigger          | Triggers workflow on a schedule                       | None                           | Define Target URLs                         | ## 1. Capture a Snapshot Using Firecrawl: Update Define blocks for URLs and update criteria. Current targets demo dynamic sites.       |
| Define Target URLs      | set                      | Defines list of target URLs as JSON array             | Schedule Trigger               | Split Out                                  |                                                                                                                                        |
| Split Out              | splitOut                 | Splits target URLs into individual items               | Define Target URLs             | Define What Matters To You                  |                                                                                                                                        |
| Define What Matters To You | set                    | Sets criteria defining meaningful website changes      | Split Out                     | Scrape Target URLs                          |                                                                                                                                        |
| Scrape Target URLs      | Firecrawl scrape          | Scrapes website content                                | Define What Matters To You     | Capture Screenshots, Get Last Snapshots    |                                                                                                                                        |
| Capture Screenshots     | Firecrawl scrape          | Captures full-page screenshots                         | Scrape Target URLs             | Save Snapshots, Get Last Snapshots          |                                                                                                                                        |
| Save Snapshots          | Notion databasePage       | Saves snapshot pages with URLs and screenshots        | Capture Screenshots            | Construct Notion Blocks                     | ## 2. Save snapshots to Notion database, splitting content into blocks below 2,000 chars using template Notion page link.              |
| Construct Notion Blocks | code                     | Splits content into blocks for Notion append           | Save Snapshots, Scrape Target URLs | Split Out Blocks                          |                                                                                                                                        |
| Split Out Blocks        | splitOut                 | Splits blocks array into individual blocks             | Construct Notion Blocks        | Append Blocks                              |                                                                                                                                        |
| Append Blocks           | Notion block              | Appends text blocks to snapshot pages                  | Split Out Blocks              | None                                       |                                                                                                                                        |
| Get Last Snapshots      | Notion databasePage       | Retrieves previous snapshots for comparison            | Capture Screenshots            | Get Many Blocks                            |                                                                                                                                        |
| Get Many Blocks         | Notion block              | Retrieves content blocks under last snapshot pages     | Get Last Snapshots             | Combine Website Data                       |                                                                                                                                        |
| Combine Website Data    | code                     | Combines current and last snapshot data per URL        | Get Many Blocks, Get Last Snapshots, Scrape Target URLs, Capture Screenshots | Split Out Data                             |                                                                                                                                        |
| Split Out Data          | splitOut                 | Splits combined website data into individual items     | Combine Website Data           | Compare Snapshots                          |                                                                                                                                        |
| Compare Snapshots       | OpenAI (LangChain)        | Uses GPT-5-Mini to detect meaningful changes           | Split Out Data                | Combine Snapshots and AI Analysis           |                                                                                                                                        |
| Combine Snapshots and AI Analysis | code             | Filters for meaningful updates excluding "NO_CHANGE"  | Compare Snapshots, Split Out Data | Split Out Updates                          |                                                                                                                                        |
| Split Out Updates       | splitOut                 | Splits updates array into individual updates           | Combine Snapshots and AI Analysis | Create Updates, Send Email Updates          |                                                                                                                                        |
| Create Updates          | Notion databasePage       | Creates update entries in Notion "Updates" database    | Split Out Updates             | Construct Update Pages                      | ## 4. Create update pages with side-by-side screenshots and summary via custom Notion API calls (columns not supported in official node). |
| Construct Update Pages  | httpRequest (Notion API)  | Adds detailed blocks (columns with images and text)    | Create Updates                | None                                       |                                                                                                                                        |
| Send Email Updates      | Gmail (OAuth2)            | Sends formatted email alerts for detected updates      | Split Out Updates             | None                                       | ## 5. Optional email alerts for updates; can be removed if not needed.                                                                  |
| Sticky Note6            | stickyNote                | Explanation and setup instructions                      | None                         | None                                       | ## How it works: Daily monitoring with Firecrawl, GPT-5 analysis, Notion storage, and optional Gmail alerts. Setup steps and contact info. |
| Sticky Note7            | stickyNote                | Notes on initial capture phase                           | None                         | None                                       | ## 1. Capture a Snapshot Using Firecrawl: update Define blocks for URLs and criteria.                                                   |
| Sticky Note8            | stickyNote                | Notes on saving snapshots to Notion                      | None                         | None                                       | ## 2. Save the Snapshot to the Notion Database: uses two Notion databases, splits content into blocks under 2,000 chars.                |
| Sticky Note9            | stickyNote                | Notes on comparing snapshots                             | None                         | None                                       | ## 3. Compare the Snapshot with the Last Snapshot If Available                                                                        |
| Sticky Note10           | stickyNote                | Notes on creating update pages                           | None                         | None                                       | ## 4. Create an Update on Notion When Changes Are Detected: uses custom Notion API calls for columns.                                   |
| Sticky Note11           | stickyNote                | Notes on optional email alerts                           | None                         | None                                       | ## 5. Optional Send an Email Alert: can be removed if not needed.                                                                       |

---

## 4. Reproducing the Workflow from Scratch

Follow these steps to rebuild the workflow manually in n8n:

1. **Create a Schedule Trigger**  
   - Node Type: scheduleTrigger  
   - Set interval to daily (or desired frequency)  

2. **Add a Set Node for Target URLs**  
   - Node Name: Define Target URLs  
   - Type: set  
   - Add a JSON output field named `target_urls` with an array of website URLs to monitor (e.g., ["https://randomtextgenerator.com", "https://randomuser.me/"])  

3. **Add a Split Out Node**  
   - Node Name: Split Out  
   - Type: splitOut  
   - Configure to split the field `target_urls`  

4. **Add a Set Node for Change Criteria**  
   - Node Name: Define What Matters To You  
   - Type: set  
   - Add a string field `change_criteria` with text specifying what counts as a meaningful change, e.g., "Either a visible text change or an image source update should be considered a change"  

5. **Add Firecrawl Scrape Node for Content**  
   - Node Name: Scrape Target URLs  
   - Type: Firecrawl (scrape)  
   - Configure URL to `={{ $json.target_urls }}`  
   - Set wait time to 5000ms  
   - Leave onlyMainContent false  
   - Use your Firecrawl API credentials  

6. **Add Firecrawl Scrape Node for Screenshots**  
   - Node Name: Capture Screenshots  
   - Type: Firecrawl (scrape)  
   - Configure URL to `={{ $json.target_urls }}`  
   - Set to capture full-page screenshot (format type: screenshot, fullPage: true)  
   - Wait 5000ms  
   - Use Firecrawl credentials  

7. **Add Notion Node to Save Snapshots**  
   - Node Name: Save Snapshots  
   - Type: Notion (databasePage)  
   - Choose your Notion Snapshots database ID  
   - Map properties:  
     - Name: `={{ new Date().toISOString().split('T')[0] + '_' + $json.target_urls.replace(/^https?:\/\//, '') }}`  
     - Screenshot: file URL from Firecrawl screenshot data  
     - URL: current URL  
     - Created Date: current date (ISO format, date only)  
   - Use your Notion API credentials  

8. **Add Code Node to Construct Notion Blocks**  
   - Node Name: Construct Notion Blocks  
   - Type: code  
   - Copy JavaScript logic to chunk scraped content into 1500-char blocks and associate with page IDs  
   - Inputs: from Save Snapshots and Scrape Target URLs  

9. **Add Split Out Node for Blocks**  
   - Node Name: Split Out Blocks  
   - Type: splitOut  
   - Split field `blocks`  

10. **Add Notion Node to Append Blocks**  
    - Node Name: Append Blocks  
    - Type: Notion (block)  
    - Block ID: dynamic from block's pageId  
    - Text content: dynamic chunk content  
    - Notion API credentials  

11. **Add Notion Node to Get Last Snapshots**  
    - Node Name: Get Last Snapshots  
    - Type: Notion (databasePage)  
    - Filter by URL equals current URL and Created Date on or before yesterday  
    - Limit 1, sort by Created Date descending  
    - Notion API credentials  

12. **Add Notion Node to Get Many Blocks**  
    - Node Name: Get Many Blocks  
    - Type: Notion (block)  
    - Block ID: from last snapshot page ID  
    - Return all child blocks  
    - Notion API credentials  

13. **Add Code Node to Combine Website Data**  
    - Node Name: Combine Website Data  
    - Type: code  
    - Combine current scrape, current screenshot, last snapshot content, last screenshot, and dates per URL into one object  

14. **Add Split Out Node for Combined Data**  
    - Node Name: Split Out Data  
    - Type: splitOut  
    - Split field `websites`  

15. **Add OpenAI Node for Comparison**  
    - Node Name: Compare Snapshots  
    - Type: OpenAI (LangChain)  
    - Model: gpt-5-mini  
    - Prompt includes: URL, current and last snapshot dates, change criteria, current content, previous content  
    - Credentials: OpenAI API key  

16. **Add Code Node to Combine AI Results**  
    - Node Name: Combine Snapshots and AI Analysis  
    - Type: code  
    - Filter out "NO_CHANGE" results and prepare update objects with summaries  

17. **Add Split Out Node for Updates**  
    - Node Name: Split Out Updates  
    - Type: splitOut  
    - Split field `updates`  

18. **Add Notion Node to Create Updates**  
    - Node Name: Create Updates  
    - Type: Notion (databasePage)  
    - Database ID: your Updates database  
    - Properties: Name with date and URL, Created At date, URL property  
    - Notion API credentials  

19. **Add HTTP Request Node to Construct Update Pages**  
    - Node Name: Construct Update Pages  
    - Type: HTTP Request  
    - Use PATCH method to `https://api.notion.com/v1/blocks/{{ $json.id }}/children`  
    - JSON body builds column list with two columns: current snapshot and previous snapshot images and headings, plus a summary block  
    - Use Notion API credentials with proper authentication  

20. **Add Gmail Node to Send Email Alerts (Optional)**  
    - Node Name: Send Email Updates  
    - Type: Gmail (OAuth2)  
    - Recipient: update with your email  
    - Subject: includes URL domain dynamically  
    - HTML body: formatted email with current and last screenshots and summary  
    - Gmail OAuth2 credentials  

21. **Connect Nodes as per the workflow:**
    - Schedule Trigger → Define Target URLs → Split Out → Define What Matters To You → Scrape Target URLs → Capture Screenshots → Save Snapshots → Construct Notion Blocks → Split Out Blocks → Append Blocks  
    - Capture Screenshots → Get Last Snapshots → Get Many Blocks → Combine Website Data → Split Out Data → Compare Snapshots → Combine Snapshots and AI Analysis → Split Out Updates → Create Updates → Construct Update Pages  
    - Split Out Updates → Send Email Updates  

22. **Test the workflow end-to-end and then enable the schedule trigger for automated daily runs.**

---

## 5. General Notes & Resources

| Note Content                                                                                                                         | Context or Link                                                                                             |
|--------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------|
| This workflow monitors websites daily, capturing snapshots, analyzing changes via GPT-5-mini, saving to Notion, and emailing alerts. | Overview and purpose                                                                                         |
| Setup steps include Firecrawl API, OpenAI API, Notion databases (Snapshots and Updates), and optional Gmail OAuth2 credentials.        | Setup instructions inside Sticky Note6                                                                      |
| Template Notion page for Snapshots and Updates databases: [Notion Template Page](https://scoutnow.notion.site/Track-Website-Changes-2b0c56764824800a993eca79f4b10bbf) | Notion page template                                                                                        |
| For support or questions, contact: hello@scoutnow.app or Twitter: [@ScoutNowApp](https://x.com/ScoutNowApp)                            | Support contact                                                                                            |
| The custom Notion update pages use direct API calls to create column layouts because official Notion node does not support columns.   | Technical note on update page construction                                                                  |

---

**Disclaimer:** The provided content is derived exclusively from an automated workflow created with n8n. It complies strictly with content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.

---