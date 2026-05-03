Generate Personalized Cold Email Icebreakers from Websites with GPT-4 & Google Sheets

https://n8nworkflows.xyz/workflows/generate-personalized-cold-email-icebreakers-from-websites-with-gpt-4---google-sheets-9882


# Generate Personalized Cold Email Icebreakers from Websites with GPT-4 & Google Sheets

### 1. Workflow Overview

This workflow automates the generation of personalized cold email icebreakers for sales outreach by scraping websites linked to lead data stored in Google Sheets. It leverages GPT-4 AI models to analyze website content and craft customized introductory messages, aiming to increase engagement rates in cold email campaigns.

The workflow is logically divided into the following functional blocks:

- **1.1 Input Reception:** Retrieves lead data from a Google Sheet, filtering for leads with specific statuses.
- **1.2 Website Scraping and Link Extraction:** Downloads the main website and extracts internal page URLs, cleaning and normalizing these links.
- **1.3 Page Content Retrieval:** Requests HTML content for each collected URL and converts it to Markdown format.
- **1.4 Content Summarization:** Uses GPT-4 to summarize each page’s content into a structured abstract.
- **1.5 Icebreaker Generation:** Aggregates all page summaries and prompts GPT-4 to generate personalized cold email icebreakers.
- **1.6 Output Update:** Writes the generated icebreakers back to the Google Sheet, updating lead statuses accordingly.
- **1.7 Error Handling:** Detects failures in scraping or content extraction and marks leads with error status for follow-up.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
This block fetches lead records from a Google Sheet, filtering leads with a specific "Status" to process individually.

**Nodes Involved:**  
- Get Search URL  
- When clicking ‘Test workflow’

**Node Details:**  

- **When clicking ‘Test workflow’**  
  - *Type:* Manual Trigger  
  - *Role:* Initiates the workflow manually for testing or controlled runs.  
  - *Configuration:* No parameters; triggers downstream data fetching.  
  - *Inputs:* None  
  - *Outputs:* Connects to "Get Search URL"  
  - *Edge Cases:* None specific; manual trigger requires user action.

- **Get Search URL**  
  - *Type:* Google Sheets (Read)  
  - *Role:* Retrieves lead data from a specified sheet, filtering where "Status" equals a desired value (likely 'Pending' or similar).  
  - *Configuration:*  
    - Document ID and sheet name set to a spreadsheet containing marketing leads.  
    - Filter: Status column = specified value (equals condition).  
    - Returns first match per execution to avoid bulk processing.  
    - Uses OAuth2 credentials linked to Google account.  
  - *Key expressions:* Filtering with `Status` column.  
  - *Inputs:* Trigger from manual start.  
  - *Outputs:* Lead record JSON including website URL and contact info.  
  - *Edge Cases:*  
    - Empty or missing rows cause workflow to stop or skip.  
    - Google Sheets API rate limits or authentication errors.  
    - Missing or malformed URLs.

---

#### 2.2 Website Scraping and Link Extraction

**Overview:**  
Downloads the lead’s main website home page, extracts all anchor links, cleans the list by removing irrelevant or external URLs, and normalizes them for further scraping.

**Nodes Involved:**  
- Scrape Home  
- HTML  
- If  
- Google Sheets1 (Error logging)  
- Edit Fields  
- Split Out  
- Code  
- Limit

**Node Details:**  

- **Scrape Home**  
  - *Type:* HTTP Request  
  - *Role:* Fetch homepage HTML content using the lead’s website URL.  
  - *Configuration:*  
    - URL from lead’s `organization_website_url`.  
    - Allows redirects.  
    - On error, continues with error output path.  
  - *Inputs:* Lead JSON from "Get Search URL".  
  - *Outputs:* Raw HTML page content or error.  
  - *Edge Cases:*  
    - Invalid URL, connection timeout, SSL errors cause errors.  
    - Non-responding or blocked sites.

- **HTML**  
  - *Type:* HTML Extractor  
  - *Role:* Parses the homepage HTML to extract all anchor (`<a>`) tag href attributes as an array.  
  - *Configuration:*  
    - Extracts `href` attributes from all `<a>` elements.  
    - Trims and cleans text.  
  - *Inputs:* Homepage HTML from "Scrape Home".  
  - *Outputs:* JSON with list of links under `links` key.  
  - *Edge Cases:*  
    - Empty or malformed HTML returns empty or error.  
    - No links found.

- **If**  
  - *Type:* Conditional Check  
  - *Role:* Checks if the extracted links array is empty to determine error handling path.  
  - *Configuration:*  
    - Condition: `links` field is empty array.  
  - *Inputs:* Output from "HTML".  
  - *Outputs:*  
    - True: Leads to error logging node "Google Sheets1".  
    - False: Continues with link processing.  
  - *Edge Cases:* Handles empty link lists gracefully.

- **Google Sheets1 (Error Logging)**  
  - *Type:* Google Sheets (Append/Update)  
  - *Role:* Updates lead record with "Error" status if homepage scraping or link extraction fails.  
  - *Configuration:*  
    - Matches lead by `linkedin_url` column.  
    - Sets `Status` field to "Error".  
  - *Inputs:* Triggered when "If" condition true.  
  - *Outputs:* Loops back to "Get Search URL" for next lead.  
  - *Edge Cases:* Google Sheets write errors; data mismatches.

- **Edit Fields**  
  - *Type:* Set (Data Transformation)  
  - *Role:* Prepares the extracted links array for splitting into individual items for processing.  
  - *Configuration:*  
    - Assigns `links` field as an array from previous node’s `links`.  
  - *Inputs:* From "If" false branch.  
  - *Outputs:* Passes array to "Split Out".  
  - *Edge Cases:* Empty arrays handled by previous node.

- **Split Out**  
  - *Type:* Split Out  
  - *Role:* Splits the array of links into individual JSON items for iterative processing.  
  - *Inputs:* Array of links from "Edit Fields".  
  - *Outputs:* Individual link items for next processing.  
  - *Edge Cases:* Empty arrays skipped.

- **Code**  
  - *Type:* Function (JavaScript)  
  - *Role:* Cleans and normalizes URLs:  
    - Removes anchors, mailto/tel links, social media links, empty or malformed links.  
    - Filters to only internal links on the same domain as the homepage.  
    - Normalizes trailing slashes and duplicates.  
    - Adds the homepage root URL explicitly.  
  - *Key Logic:*  
    - Extracts base domain from homepage URL.  
    - Checks each link’s domain matches base domain.  
    - Returns a cleaned list of unique internal URLs.  
  - *Inputs:* Individual links from "Split Out"  
  - *Outputs:* Cleaned, unique internal URLs as individual items.  
  - *Edge Cases:* Unexpected URL formats may cause misses; malformed input throws error.

- **Limit**  
  - *Type:* Limit  
  - *Role:* Limits number of URLs processed further per lead to 3 (configurable).  
  - *Configuration:* Max items = 3  
  - *Inputs:* Cleaned URLs from "Code" node.  
  - *Outputs:* Limited set of URLs for content fetching.  
  - *Edge Cases:* Limits processing load; fewer than 3 links processes all.

---

#### 2.3 Page Content Retrieval

**Overview:**  
Fetches HTML content for each internal URL extracted and converts HTML to Markdown for easier text summarization.

**Nodes Involved:**  
- Request web page for URL  
- Markdown

**Node Details:**  

- **Request web page for URL**  
  - *Type:* HTTP Request  
  - *Role:* Downloads full HTML content for each internal page URL.  
  - *Configuration:*  
    - URL dynamically set from cleaned links list.  
    - On error: continues regular output to avoid stopping entire workflow.  
  - *Inputs:* URLs from "Limit" node.  
  - *Outputs:* Raw HTML content of each internal page.  
  - *Edge Cases:* Timeout, 404s, or blocked pages produce errors but do not stop processing.

- **Markdown**  
  - *Type:* HTML to Markdown Converter  
  - *Role:* Converts HTML content to Markdown text format for cleaner input to GPT summarization.  
  - *Configuration:*  
    - Input HTML is the raw page content.  
    - If no data, returns placeholder `<div>empty</div>`.  
  - *Inputs:* HTML content from HTTP request.  
  - *Outputs:* Markdown text content.  
  - *Edge Cases:* Empty or malformed HTML returns minimal markdown.

---

#### 2.4 Content Summarization

**Overview:**  
Uses GPT-4 to create detailed two-paragraph abstracts summarizing the content of each page.

**Nodes Involved:**  
- Summarize Website Page  
- Aggregate

**Node Details:**  

- **Summarize Website Page**  
  - *Type:* OpenAI (GPT-4 via LangChain node)  
  - *Role:* Summarizes the Markdown content of each page into a structured JSON abstract.  
  - *Configuration:*  
    - Model: GPT-4.1 variant.  
    - System prompt defines role as a website scraping assistant.  
    - User prompt requests a two-paragraph comprehensive abstract in JSON format with key "abstract".  
    - If input is empty, returns "no content".  
  - *Inputs:* Markdown text from "Markdown" node.  
  - *Outputs:* JSON with page abstract.  
  - *Credentials:* OpenAI API key configured.  
  - *Edge Cases:*  
    - API timeouts, rate limits, or invalid input cause failure.  
    - Empty page handled gracefully.

- **Aggregate**  
  - *Type:* Aggregate  
  - *Role:* Collects all page abstracts from the current lead into a single aggregated array for final icebreaker generation.  
  - *Configuration:* Aggregates on the field `message.content.abstract` from summaries.  
  - *Inputs:* Summaries from "Summarize Website Page".  
  - *Outputs:* Single JSON containing all abstracts combined.  
  - *Edge Cases:* Empty aggregation returns empty array.

---

#### 2.5 Icebreaker Generation

**Overview:**  
Processes the aggregated page summaries with GPT-4 to generate a tailored cold email icebreaker, formatted as JSON.

**Nodes Involved:**  
- Generate Multiline Icebreaker  
- Google Sheets

**Node Details:**  

- **Generate Multiline Icebreaker**  
  - *Type:* OpenAI (GPT-4 via LangChain node)  
  - *Role:* Creates a personalized icebreaker message based on the collected website abstracts, designed for cold email outreach.  
  - *Configuration:*  
    - Model: GPT-4.1.  
    - Temperature: 0.5 for balanced creativity.  
    - System prompt sets role as intelligent sales assistant.  
    - User prompt includes detailed instructions to:  
      - Write spartan, laconic tone.  
      - Use non-obvious, personalized details from website summaries.  
      - Format output strictly in JSON with key "icebreaker".  
      - Avoid clichés and emojis.  
    - Variables like company name, location, and implied beliefs used to tailor message.  
  - *Inputs:* Aggregated abstracts from "Aggregate".  
  - *Outputs:* JSON with generated icebreaker string.  
  - *Credentials:* OpenAI API key configured.  
  - *Edge Cases:* API errors or malformed input; prompt strictness may cause generation issues.

- **Google Sheets**  
  - *Type:* Google Sheets (Append/Update)  
  - *Role:* Updates lead record with generated icebreaker and marks status as "Done".  
  - *Configuration:*  
    - Matches lead by `linkedin_url`.  
    - Writes fields: `IceBreaker` (generated text), `Status` = "Done".  
    - Uses OAuth2 credentials.  
  - *Inputs:* Output from icebreaker generation.  
  - *Outputs:* Loops back to "Get Search URL" for next lead.  
  - *Edge Cases:* Google Sheets API errors, concurrency issues.

---

#### 2.6 Error Handling

**Overview:**  
Handles failures encountered during homepage scraping or link extraction by marking leads as "Error" and skipping further processing.

**Nodes Involved:**  
- If (empty links condition)  
- Google Sheets1 (Error status update)

**Node Details:**  

- Covered in 2.2 with conditional check and error update node.

---

### 3. Summary Table

| Node Name                 | Node Type                         | Functional Role                                   | Input Node(s)               | Output Node(s)            | Sticky Note                                                                                                                       |
|---------------------------|----------------------------------|--------------------------------------------------|-----------------------------|---------------------------|----------------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger                   | Manual start of the workflow                      | None                        | Get Search URL             |                                                                                                                                    |
| Get Search URL            | Google Sheets                    | Fetch lead data filtered by status                | When clicking ‘Test workflow’ | Scrape Home               |                                                                                                                                    |
| Scrape Home               | HTTP Request                    | Download homepage HTML of lead’s website          | Get Search URL               | HTML, Google Sheets1       | # Step-1: Downloads one webpage and creates list of pages/subpages; errors cause status "Error" update.                           |
| HTML                      | HTML Extractor                  | Extract all anchor href links from homepage HTML  | Scrape Home                 | If                        |                                                                                                                                    |
| If                        | Conditional                    | Checks if extracted link list is empty             | HTML                        | Google Sheets1, Edit Fields |                                                                                                                                    |
| Google Sheets1            | Google Sheets                    | Updates lead with status "Error" if scraping fails | If (true branch)             | Get Search URL             |                                                                                                                                    |
| Edit Fields               | Set                            | Prepares links array for splitting                  | If (false branch)            | Split Out                  |                                                                                                                                    |
| Split Out                 | Split Out                      | Splits links array into individual items            | Edit Fields                 | Code                      |                                                                                                                                    |
| Code                      | Function (JavaScript)           | Cleans, normalizes, and filters internal URLs       | Split Out                   | Limit                     |                                                                                                                                    |
| Limit                     | Limit                          | Limits number of URLs to process per lead to 3      | Code                        | Request web page for URL   |                                                                                                                                    |
| Request web page for URL  | HTTP Request                    | Downloads HTML content of each internal page       | Limit                       | Markdown                   | # Step-2: Visits each collected page and converts to HTML text.                                                                    |
| Markdown                  | Markdown Converter             | Converts HTML to Markdown text                        | Request web page for URL     | Summarize Website Page    |                                                                                                                                    |
| Summarize Website Page    | OpenAI (GPT-4 via LangChain)   | Summarizes page content into a detailed abstract     | Markdown                    | Aggregate                  | # Step-3: Summarizes pages to abstracts using GPT, outputs JSON.                                                                   |
| Aggregate                 | Aggregate                      | Aggregates all page abstracts into one array          | Summarize Website Page      | Generate Multiline Icebreaker |                                                                                                                                    |
| Generate Multiline Icebreaker | OpenAI (GPT-4 via LangChain)   | Generates personalized cold email icebreaker message | Aggregate                   | Google Sheets              | # Step-3: Generates personalized icebreaker from summaries.                                                                        |
| Google Sheets             | Google Sheets                  | Updates lead with generated icebreaker and status "Done" | Generate Multiline Icebreaker | Get Search URL             |                                                                                                                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger Node:**  
   - Name: "When clicking ‘Test workflow’"  
   - Purpose: Manual start for testing.

2. **Add Google Sheets Node (Get Search URL):**  
   - Operation: Read rows from lead sheet.  
   - Set Document ID and Sheet Name to your leads spreadsheet.  
   - Filter rows where `Status` equals the desired value (e.g., "Pending").  
   - Configure Google Sheets OAuth2 credentials.  
   - Connect from Manual Trigger.

3. **Add HTTP Request Node (Scrape Home):**  
   - Operation: GET request.  
   - Set URL to `{{$json["organization_website_url"]}}`.  
   - Enable redirects.  
   - On error: Continue with error output.  
   - Connect from "Get Search URL".

4. **Add HTML Extract Node (HTML):**  
   - Operation: Extract HTML content.  
   - Extract all href attributes from `<a>` tags as array.  
   - Enable trimming and cleanup.  
   - Connect from "Scrape Home".

5. **Add If Node (Empty Links Check):**  
   - Condition: Check if `links` array is empty.  
   - Connect from "HTML".

6. **Add Google Sheets Node (Google Sheets1) for Error Logging:**  
   - Operation: Append or update.  
   - Match by `linkedin_url`.  
   - Set `Status` field to "Error".  
   - Use same Google Sheets credentials.  
   - Connect from If node's "true" branch (empty links).

7. **Add Set Node (Edit Fields):**  
   - Assign `links` field as array from extracted links.  
   - Connect from If node's "false" branch.

8. **Add Split Out Node (Split Out):**  
   - Split `links` array into individual items.  
   - Connect from "Edit Fields".

9. **Add Function Node (Code):**  
   - Paste JavaScript code to:  
     - Extract base domain from homepage URL.  
     - Filter links for internal URLs only.  
     - Remove anchors, mailto, social URLs, duplicates.  
     - Normalize trailing slashes.  
     - Include homepage URL explicitly.  
   - Connect from "Split Out".

10. **Add Limit Node (Limit):**  
    - Max items: 3 (or adjust as needed).  
    - Connect from "Code".

11. **Add HTTP Request Node (Request web page for URL):**  
    - Operation: GET request.  
    - URL from cleaned links.  
    - On error: Continue with regular output.  
    - Connect from "Limit".

12. **Add Markdown Node (Markdown):**  
    - Convert HTML to Markdown.  
    - Input: HTML content from previous node.  
    - Connect from "Request web page for URL".

13. **Add OpenAI Node (Summarize Website Page):**  
    - Model: GPT-4.1.  
    - Prompt: System role as website scraping assistant.  
    - User prompt asks for two-paragraph JSON abstract of page content.  
    - Input: Markdown text.  
    - Enable JSON output.  
    - Connect from "Markdown".  
    - Configure OpenAI API credentials.

14. **Add Aggregate Node (Aggregate):**  
    - Aggregate field: `message.content.abstract`.  
    - Connect from "Summarize Website Page".

15. **Add OpenAI Node (Generate Multiline Icebreaker):**  
    - Model: GPT-4.1.  
    - Temperature: 0.5.  
    - System role: Sales assistant.  
    - User prompt instructs to generate personalized icebreaker in JSON format including variables from lead data and website summaries.  
    - Input: Aggregated abstracts.  
    - Enable JSON output.  
    - Connect from "Aggregate".  
    - Use OpenAI API credentials.

16. **Add Google Sheets Node (Google Sheets):**  
    - Operation: Append or update.  
    - Match by `linkedin_url`.  
    - Update fields:  
      - `IceBreaker` with generated message.  
      - `Status` set to "Done".  
    - Use Google Sheets OAuth2 credentials.  
    - Connect from "Generate Multiline Icebreaker".

17. **Connect Google Sheets (Error Logging) and Google Sheets (Success) nodes back to "Get Search URL"** to process next lead iteratively.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                           | Context or Link                                                                                                                         |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------|
| The workflow requires OpenAI API credentials with access to GPT-4 models. Ensure your quota and rate limits align with expected volume.                                                                                                                                                                                                                                               | OpenAI API documentation                                                                                                               |
| Google Sheets OAuth2 setup must have read and write permissions on the specified spreadsheet. The sheet must contain columns like `Status`, `linkedin_url`, `organization_website_url`, and `IceBreaker`.                                                                                                                                                                              | Google Sheets API documentation                                                                                                       |
| The "Code" node filtering logic excludes social media links and external domains to focus on internal pages only, improving relevance of summaries and icebreakers. Adjust domain filter if leads have multiple domains.                                                                                                                                                                | N/A                                                                                                                                    |
| The summarization prompt enforces a strict JSON format for easy parsing and downstream use. The icebreaker generation prompt is carefully crafted to avoid clichés and ensure personalized, concise messages.                                                                                                                                                                         | Prompts embedded in respective GPT nodes                                                                                              |
| If a lead’s website is unreachable or malformed, the workflow marks the lead as "Error" and continues with the next lead to avoid blocking.                                                                                                                                                                                                                                           | Sticky Note on Step-1                                                                                                                  |
| The workflow currently limits to processing three internal pages per website to balance depth and API usage cost. Adjust "Limit" node as needed.                                                                                                                                                                                                                                      | Sticky Note on Step-2                                                                                                                  |
| The icebreakers are designed for cold email outreach aimed at web agencies and similar businesses, emphasizing automation and AI-driven personalization.                                                                                                                                                                                                                              | Sticky Note on Step-3                                                                                                                  |
| For best results, maintain clean and accurate lead data in the Google Sheet, including valid URLs and LinkedIn URLs for matching.                                                                                                                                                                                                                                                     | N/A                                                                                                                                    |

---

**Disclaimer:**  
The text provided is exclusively derived from an automated workflow created with n8n, a tool for integration and automation. This processing strictly complies with content policies and contains no illegal, offensive, or protected materials. All data handled is legal and public.