Brand DNA Generator Using JotForm, Google Search, Gemini AI & Notion

https://n8nworkflows.xyz/workflows/brand-dna-generator-using-jotform--google-search--gemini-ai---notion-10617


# Brand DNA Generator Using JotForm, Google Search, Gemini AI & Notion

### 1. Workflow Overview

This workflow automates the generation of a comprehensive Brand DNA profile for companies by integrating data from form submissions, Google search results, AI-powered content extraction, and Notion database storage. It is designed for marketing strategists, brand managers, and growth teams who need structured brand insights without manual research.

The logical structure is divided into these blocks:

- **1.1 Input Reception and URL Normalization:** Receive company data via a JotForm submission and sanitize the website URL.
- **1.2 Domain-Based Google Search & URL Extraction:** Search for related web pages using SerpAPI based on the company domain, extracting and deduplicating relevant URLs.
- **1.3 Web Content Retrieval and Cleaning:** Fetch raw HTML content of each URL, then clean and extract readable text.
- **1.4 AI-Powered Information Extraction per Page:** Use AI to extract structured brand-relevant information from each page’s content.
- **1.5 Data Merging and Brand DNA Consolidation:** Combine multiple page-level extraction results into a unified Brand DNA record.
- **1.6 Formatting and Notion Integration:** Format the final Brand DNA JSON into text blocks and create a structured page in a Notion database.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and URL Normalization

**Overview:**  
This block triggers the workflow on new form submissions and processes the submitted company website URL to ensure a valid domain is extracted.

**Nodes Involved:**  
- JotForm Trigger  
- Code in JavaScript

**Node Details:**

- **JotForm Trigger**  
  - Type: Trigger node for JotForm form submissions  
  - Configuration: Connected to a specific form via the JotForm API credential  
  - Inputs: Incoming form submission JSON with fields including "Company Name" and "Company Website"  
  - Outputs: Form data forwarded downstream  
  - Failure Modes: API key invalid, webhook issues, missing form fields  
  - Notes: Starts the workflow upon new submission

- **Code in JavaScript**  
  - Type: Function node executing JavaScript  
  - Role: Clean and normalize "Company Website" field — trims whitespace, ensures URL protocol (https://), and extracts the domain (removes www.)  
  - Key Expression: Uses JavaScript `URL` constructor with try-catch fallback for malformed URLs  
  - Inputs: JSON from JotForm Trigger  
  - Outputs: Original data augmented with a new "Domain" field for domain name extraction  
  - Edge cases: Malformed URLs, empty or missing website field  
  - Sticky Note: "- Starts the workflow when a new form submission is received from JotForm. - Cleans and standardizes the submitted website URL."

---

#### 2.2 Domain-Based Google Search & URL Extraction

**Overview:**  
Performs a Google search limited to the extracted domain, then parses and deduplicates URLs from the search results for further content extraction.

**Nodes Involved:**  
- Google search (SerpAPI)  
- Set url (Code in JavaScript)  
- Loop Over Items (Split in Batches)

**Node Details:**

- **Google search**  
  - Type: SerpAPI search node  
  - Role: Executes a domain-specific Google query using `site:<domain>` syntax  
  - Inputs: Domain from previous step  
  - Outputs: Raw SerpAPI JSON with search results  
  - Failure Modes: API key limits, network issues, empty results  
  - Sticky Note: "- Automatically retrieves search results related to the company’s domain. - Extracts and cleans URLs from the SerpAPI search results."

- **Set url**  
  - Type: Code in JavaScript  
  - Role: Parses `organic_results` from SerpAPI response, extracts `link` or fallback URLs from redirect links, deduplicates URLs  
  - Inputs: SerpAPI search results JSON  
  - Outputs: One item per unique URL with metadata (position, title, displayed_link, etc.)  
  - Edge cases: No organic results, malformed URLs in response, empty results  
  - Notes: Returns empty placeholder item to maintain flow on no results

- **Loop Over Items**  
  - Type: SplitInBatches  
  - Role: Processes each URL one at a time downstream to avoid rate limits or overload  
  - Inputs: List of URLs from Set url  
  - Outputs: Single URL item passed downstream sequentially

---

#### 2.3 Web Content Retrieval and Cleaning

**Overview:**  
Fetches HTML content from each URL and cleans it to extract readable, plain text for AI processing.

**Nodes Involved:**  
- HTTP Request  
- Code in JavaScript1  
- WebpageContentExtractor

**Node Details:**

- **HTTP Request**  
  - Type: HTTP Request node  
  - Role: Downloads raw HTML content from the URL  
  - Inputs: URL from Loop Over Items  
  - Outputs: Raw HTML in response body  
  - Failure Modes: HTTP errors, timeouts, redirects, non-200 responses  
  - Sticky Note: "- Fetches raw HTML content from a given website URL. - Clean and extract readable text from raw HTML."

- **Code in JavaScript1**  
  - Type: Code node  
  - Role: Cleans HTML by removing tags, scripts, styles, comments, and entities, leaving plain text  
  - Key Logic: Uses regex replacements for HTML cleaning and decoding entities  
  - Inputs: Raw HTML from HTTP Request  
  - Outputs: Cleaned plain text in `data` field  
  - Edge cases: Malformed HTML, empty content

- **WebpageContentExtractor**  
  - Type: Webpage Content Extractor node  
  - Role: Further extracts and structures readable content from cleaned HTML text  
  - Inputs: Cleaned text from Code in JavaScript1  
  - Outputs: Structured textual content for AI extraction  
  - On Error: Continues flow even if extraction fails (prevents blocking)  
  - Sticky Note: "- Extracts structured and readable content from raw HTML. - Uses AI to extract structured company-related information from the article."

---

#### 2.4 AI-Powered Information Extraction per Page

**Overview:**  
Uses a language model to extract structured, relevant brand information from each web page’s cleaned content.

**Nodes Involved:**  
- Information Extractor  
- OpenRouter Chat Model

**Node Details:**

- **Information Extractor**  
  - Type: Langchain Information Extractor node  
  - Role: Formats prompt with company name and webpage text, instructs the LLM to extract relevant brand info in JSON  
  - Configuration: System prompt emphasizes factual extraction without fabrication, expects JSON output schema for fields like companyName, relatedToBrand, isRelated, relationType, reason, articleGoal, excerpt, summary, keyPoints  
  - Inputs: Cleaned webpage content + company name from JotForm Trigger  
  - Outputs: JSON extraction result per page  
  - Failure Modes: LLM timeout, invalid JSON response, prompt errors

- **OpenRouter Chat Model**  
  - Type: Langchain chat LLM node (OpenRouter backend)  
  - Role: Provides the AI model (Google Gemini 2.0 flash experimental free tier) for Information Extractor  
  - Inputs: Prompt from Information Extractor node  
  - Outputs: LLM-generated JSON response  
  - Notes: Ensures AI-powered extraction is integrated

---

#### 2.5 Data Merging and Brand DNA Consolidation

**Overview:**  
Aggregates multiple page-level extraction results into a single comprehensive Brand DNA record, merging lists, deduplicating, and consolidating fields.

**Nodes Involved:**  
- Merge Data (Code in JavaScript)  
- Information Extractor1  
- OpenRouter Chat Model1

**Node Details:**

- **Merge Data**  
  - Type: Code node in JavaScript  
  - Role: Combines multiple extraction JSON objects into one; merges arrays with deduplication, merges scalar fields preferring the first non-empty value, sets a boolean flag if any page is related  
  - Inputs: Array of extraction results JSON  
  - Outputs: Single combined JSON object representing aggregated Brand DNA data  
  - Edge cases: Empty inputs, inconsistent field types, partial data

- **Information Extractor1**  
  - Type: Langchain Information Extractor node  
  - Role: Acts as a brand strategist AI to merge multiple extraction results into a concise Brand DNA JSON profile  
  - Configuration: System prompt instructs strict JSON output, factual consolidation, removing duplicates and overlap, neutral language  
  - Inputs: Output of Merge Data with fields like companyName, relatedToBrand, relationType, reason, articleGoal, excerpt, summary, keyPoints  
  - Outputs: Brand DNA candidate JSON object with fields like companyDescription, idealCustomerProfile, painPoints, valueProposition, proofPoints, reasonsToOutreach, outreachChannels, suggestedKeywords, brandTone  
  - Failure Modes: LLM errors, invalid JSON, missing or incomplete fields

- **OpenRouter Chat Model1**  
  - Type: Langchain chat LLM node (OpenRouter backend)  
  - Role: Provides LLM for Information Extractor1 to synthesize final Brand DNA  
  - Inputs: Prompt from Information Extractor1  
  - Outputs: Final Brand DNA JSON

- Sticky Note: "- Combine multiple extraction results (from many pages) into a single deduplicated record. - Consolidate and synthesize extracted article-level data into a Brand DNA candidate. - Provide the language model (LLM) backend used by the Information Extractor node."

---

#### 2.6 Formatting and Notion Integration

**Overview:**  
Transforms the final Brand DNA JSON into formatted text blocks suitable for Notion, then creates a new page in a specified Notion database containing all brand insights.

**Nodes Involved:**  
- Code in JavaScript2  
- Create a database page (Notion)

**Node Details:**

- **Code in JavaScript2**  
  - Type: Code node  
  - Role: Converts Brand DNA JSON fields into separate text fields formatted for Notion blocks, including headings and bullet-point paragraphs, each limited to ~1900 characters (safe Notion block size)  
  - Logic:  
    - Headings (h1, h2_xxx) for section titles  
    - Paragraphs (p_xxx) with bulleted lists or scalar text  
    - Fallback values for empty fields with "- (none)"  
  - Inputs: Brand DNA JSON from Information Extractor1  
  - Outputs: Formatted text fields for Notion  
  - Edge cases: Very long text truncated, missing fields handled gracefully

- **Create a database page (Notion)**  
  - Type: Notion node (databasePage resource)  
  - Role: Creates a new page in the configured Notion database with the formatted Brand DNA content as structured blocks (headings and paragraphs)  
  - Configuration: Uses Notion Integration Token, databaseId provided, block UI configured with dynamic content fields from previous node  
  - Inputs: Formatted strings from Code in JavaScript2  
  - Outputs: Confirmation of page creation  
  - Failure Modes: Invalid token, databaseId, API limits, formatting errors  
  - Sticky Note: "- Format consolidated Brand DNA JSON into string fields suitable for Notion blocks. - Persist the formatted Brand DNA into a Notion database as a new page."

---

### 3. Summary Table

| Node Name               | Node Type                           | Functional Role                                   | Input Node(s)              | Output Node(s)             | Sticky Note                                                                                                           |
|-------------------------|-----------------------------------|--------------------------------------------------|----------------------------|----------------------------|-----------------------------------------------------------------------------------------------------------------------|
| JotForm Trigger         | jotFormTrigger                    | Trigger on new form submission                    | (trigger)                  | Code in JavaScript          |                                                                                                                       |
| Code in JavaScript      | code                             | Normalize and extract domain from website URL    | JotForm Trigger            | Google search               | - Starts the workflow when a new form submission is received from JotForm. - Cleans and standardizes the submitted website URL. |
| Google search           | serpApi                          | Search Google for pages related to domain         | Code in JavaScript         | Set url                    | - Automatically retrieves search results related to the company’s domain. - Extracts and cleans URLs from the SerpAPI search results. |
| Set url                 | code                             | Extract and deduplicate URLs from search results  | Google search              | Loop Over Items             |                                                                                                                       |
| Loop Over Items         | splitInBatches                   | Process URLs sequentially                          | Set url                    | Merge Data, HTTP Request    |                                                                                                                       |
| HTTP Request            | httpRequest                      | Download raw HTML content from URL                | Loop Over Items            | Code in JavaScript1         | - Fetches raw HTML content from a given website URL. - Clean and extract readable text from raw HTML.                  |
| Code in JavaScript1     | code                             | Clean raw HTML to plain text                       | HTTP Request               | WebpageContentExtractor     |                                                                                                                       |
| WebpageContentExtractor | webpageContentExtractor          | Extract structured readable content from HTML    | Code in JavaScript1        | Information Extractor       | - Extracts structured and readable content from raw HTML. - Uses AI to extract structured company-related information from the article. |
| Information Extractor   | informationExtractor (Langchain) | Extract brand info JSON from page content         | WebpageContentExtractor    | Loop Over Items             |                                                                                                                       |
| OpenRouter Chat Model   | lmChatOpenRouter                 | LLM backend to support Information Extractor     | Information Extractor      | (ai_languageModel connection) |                                                                                                                       |
| Merge Data              | code                             | Combine multiple extraction results into one      | Loop Over Items            | Information Extractor1      | - Combine multiple extraction results (from many pages) into a single deduplicated record. - Consolidate and synthesize extracted article-level data into a Brand DNA candidate. - Provide the language model (LLM) backend used by the Information Extractor node. |
| Information Extractor1  | informationExtractor (Langchain) | Merge multiple extraction results into Brand DNA | Merge Data                 | Code in JavaScript2         |                                                                                                                       |
| OpenRouter Chat Model1  | lmChatOpenRouter                 | LLM backend to support Information Extractor1    | Information Extractor1     | (ai_languageModel connection) |                                                                                                                       |
| Code in JavaScript2     | code                             | Format Brand DNA JSON into Notion text blocks     | Information Extractor1     | Create a database page      | - Format consolidated Brand DNA JSON into string fields suitable for Notion blocks. - Persist the formatted Brand DNA into a Notion database as a new page. |
| Create a database page  | notion                           | Create a Notion page with Brand DNA content       | Code in JavaScript2        |                            |                                                                                                                       |
| Sticky Note             | stickyNote                      | Documentation and overview                         |                            |                            | # Automated Brand DNA Generator Using JotForm, Google Search, AI Extraction & Notion ... (full content in node)         |
| Sticky Note1            | stickyNote                      | Notes on JotForm Trigger and URL normalization    |                            |                            | - Starts the workflow when a new form submission is received from JotForm. - Cleans and standardizes the submitted website URL. |
| Sticky Note2            | stickyNote                      | Notes on Google Search and URL extraction          |                            |                            | - Automatically retrieves search results related to the company’s domain. - Extracts and cleans URLs from the SerpAPI search results. |
| Sticky Note3            | stickyNote                      | Notes on data merging and AI consolidation          |                            |                            | - Combine multiple extraction results (from many pages) into a single deduplicated record. - Consolidate and synthesize extracted article-level data into a Brand DNA candidate. - Provide the language model (LLM) backend used by the Information Extractor node. |
| Sticky Note4            | stickyNote                      | Notes on formatting and Notion integration          |                            |                            | - Format consolidated Brand DNA JSON into string fields suitable for Notion blocks. - Persist the formatted Brand DNA into a Notion database as a new page. |
| Sticky Note5            | stickyNote                      | Notes on HTTP Request and HTML cleaning             |                            |                            | - Fetches raw HTML content from a given website URL. - Clean and extract readable text from raw HTML.                  |
| Sticky Note6            | stickyNote                      | Notes on content extraction and AI analysis         |                            |                            | - Extracts structured and readable content from raw HTML. - Uses AI to extract structured company-related information from the article. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create JotForm Trigger Node**  
   - Type: JotForm Trigger  
   - Credentials: Connect your JotForm account with API key  
   - Select the form that includes "Company Name" and "Company Website" fields.

2. **Add Code Node to Normalize Website URL**  
   - Type: Code (JavaScript)  
   - Paste script to trim website input, add https:// if missing, extract domain removing www.  
   - Connect input from JotForm Trigger.

3. **Add SerpAPI Google Search Node**  
   - Type: SerpAPI  
   - Credentials: Add your SerpAPI Key  
   - Query set to `site:{{ $json.Domain }}` to restrict search to domain  
   - Input connected from Code Node above.

4. **Add Code Node to Extract and Deduplicate URLs**  
   - Type: Code (JavaScript)  
   - Paste code to parse `organic_results`, extract `link` or redirect URLs, deduplicate  
   - Connect input from SerpAPI node.

5. **Add SplitInBatches Node**  
   - Type: SplitInBatches  
   - No special options required  
   - Connect input from URL extraction code node.

6. **Add HTTP Request Node to Fetch Webpage HTML**  
   - Type: HTTP Request  
   - URL parameter set to `{{$json.url}}` (dynamic URL from batch)  
   - Connect input from SplitInBatches node.

7. **Add Code Node to Clean HTML**  
   - Type: Code (JavaScript)  
   - Paste code that strips HTML tags, scripts, styles, comments, and decodes entities to get clean text  
   - Input from HTTP Request node.

8. **Add WebpageContentExtractor Node**  
   - Type: Webpage Content Extractor  
   - Input HTML parameter set to `{{$json.data}}` (cleaned text from previous node)  
   - Set onError to continue to avoid blocking.

9. **Add Langchain Information Extractor Node**  
   - Type: Langchain - Information Extractor  
   - Configure prompt template with company name and webpage content, instructing strict factual JSON extraction  
   - Connect input from WebpageContentExtractor  
   - Attach OpenRouter Chat Model node (Google Gemini 2.0 flash experimental free tier) as AI model backend.

10. **Connect Information Extractor Output to SplitInBatches Node**  
    - After extraction, connect output back to SplitInBatches node to process next URL.

11. **Add Code Node to Merge Multiple Extraction Results**  
    - Type: Code (JavaScript)  
    - Paste code that combines multiple JSON objects into a single deduplicated Brand DNA record, merging arrays and scalar fields  
    - Connect input from SplitInBatches node's first output (merging path).

12. **Add Second Langchain Information Extractor Node for Brand DNA Synthesis**  
    - Type: Langchain - Information Extractor  
    - Configure prompt as brand strategist to merge and condense multiple extraction results into final Brand DNA JSON  
    - Connect input from Merge Data code node  
    - Attach second OpenRouter Chat Model node as AI backend.

13. **Add Code Node to Format Brand DNA for Notion**  
    - Type: Code (JavaScript)  
    - Paste code that converts Brand DNA JSON fields into Notion block-formatted strings (headings and bullet lists) with character limits  
    - Connect input from Information Extractor1 node.

14. **Add Notion Node to Create Database Page**  
    - Type: Notion (databasePage resource)  
    - Credentials: Connect your Notion Integration Token  
    - Set databaseId to target Notion database  
    - Configure block UI with dynamic content fields from previous formatting node (headings and paragraphs)  
    - Connect input from formatting code node.

15. **Add Sticky Notes for Documentation** (optional)  
    - Add sticky notes describing each block’s purpose and instructions for clarity.

16. **Activate Workflow and Test**  
    - Submit test data through the configured JotForm form  
    - Monitor execution and verify data appears correctly in Notion.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    | Context or Link                                                                                          |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| The Brand DNA Generator workflow automatically scans and analyzes online content to build a company’s Brand DNA profile. It uses form inputs, Google search scraping, AI-powered extraction, and Notion integration to deliver structured brand insights without manual research.                                                                                                                                                                                                                                                      | Overview from primary sticky note                                                                              |
| Requires API keys for JotForm, SerpAPI, OpenRouter (or compatible LLM service), and Notion Integration Token with database access.                                                                                                                                                                                                                                                                                                                                                                                                    | Setup requirements                                                                                         |
| AI model used is Google Gemini 2.0 Flash experimental via OpenRouter free tier. Prompts are carefully crafted for factual, strict JSON extraction without fabrication.                                                                                                                                                                                                                                                                                                                                                                    | AI model and prompt design                                                                                 |
| Handles edge cases gracefully: malformed URLs, empty search results, HTTP failures, and LLM errors with fallback and continuation to avoid blocking the workflow.                                                                                                                                                                                                                                                                                                                                                                      | Robustness notes                                                                                           |
| Final output saved in Notion database with sections like Company Description, Ideal Customer Profile, Pain Points, Value Proposition, Proof Points, Reasons to Outreach, Outreach Channels, Suggested Keywords, Brand Tone.                                                                                                                                                                                                                                                                                                               | Final delivered Brand DNA report format                                                                    |
| For more details and setup instructions, refer to the sticky note content included in the workflow nodes.                                                                                                                                                                                                                                                                                                                                                                                                                                     | Internal documentation embedded in workflow sticky notes                                                  |

---

**Disclaimer:**  
The text provided here is exclusively derived from an automated n8n workflow. It fully complies with content policies and handles only legal, public data. No illegal or offensive content is involved.