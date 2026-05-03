Generate SEO Content with Claude AI & Competitor Analysis using Apify

https://n8nworkflows.xyz/workflows/generate-seo-content-with-claude-ai---competitor-analysis-using-apify-8946


# Generate SEO Content with Claude AI & Competitor Analysis using Apify

### 1. Workflow Overview

This workflow automates the generation of SEO content by combining AI-driven meta tag and content brief creation with competitor analysis scraped from Google search results. It is designed to intake client and SEO data from Google Sheets, perform competitor research using Apify and Firecrawl scraping services, analyze competitor page structures, and then generate optimized meta titles, descriptions, and H1 tags with Claude AI (Anthropic Claude Sonnet 4 model). Finally, it creates detailed SEO content briefs and updates the original Google Sheets with all generated SEO elements.

The workflow is structured into five main logical blocks:

- **1.1 Setup and Configuration**: Receive input, load client and SEO sheet data, validate keywords, and prepare for batch processing.
- **1.2 Competitor Research and Analysis**: Use Apify API to fetch competitor URLs and Firecrawl to scrape competitor headings (H1-H6).
- **1.3 Meta Tags and H1 Generation**: Generate SEO-optimized meta titles, descriptions, and H1 tags using Claude AI with competitor data context.
- **1.4 Content Brief Creation**: Use Claude AI to produce a detailed SEO content brief analyzing search intent and competitor content.
- **1.5 Data Integration and Sheet Updates**: Compile generated content and update the Google Sheets document accordingly, maintaining batch processing for multiple keywords.

---

### 2. Block-by-Block Analysis

#### 1.1 Setup and Configuration

- **Overview:**  
  This block initializes the workflow by receiving a Google Sheets URL via a chat trigger, reading client and SEO sheet data, filtering rows based on keyword presence and H1 absence, and preparing the data for batch processing.

- **Nodes Involved:**  
  - When chat message received  
  - Client Information  
  - SEO information  
  - Filter1  
  - If1  
  - Loop Over Items  

- **Node Details:**  

  - **When chat message received**  
    - Type: Langchain Chat Trigger (webhook)  
    - Role: Entry point that listens for a chat input containing the Google Sheets URL.  
    - Config: Public webhook; uses response node mode to send replies back.  
    - Inputs: HTTP webhook request with chatInput field containing the Google Sheets URL.  
    - Outputs: Passes JSON including the chatInput URL downstream.  
    - Edge cases: Missing or malformed URL input; webhook availability must be ensured.

  - **Client Information**  
    - Type: Google Sheets node  
    - Role: Reads the "Client information" sheet from the provided Google Sheets URL.  
    - Config: Reads all rows from sheet named "Client information" in the spreadsheet URL passed from chat trigger.  
    - Inputs: Receives Google Sheets URL from trigger.  
    - Outputs: JSON data including client name, description, URL, tone, restrictions.  
    - Edge cases: Invalid or inaccessible sheet URL; sheet not found; authentication errors.

  - **SEO information**  
    - Type: Google Sheets node  
    - Role: Reads the "SEO" sheet from the same Google Sheets document.  
    - Config: Reads all rows from sheet named "SEO".  
    - Inputs: Google Sheets URL from trigger.  
    - Outputs: JSON data including keyword, page info, awareness, page type.  
    - Edge cases: Same as Client Information node.

  - **Filter1**  
    - Type: Filter node  
    - Role: Filters SEO rows to keep only those where Keyword is not empty and <h1> field is empty.  
    - Config: AND conditions: Keyword not empty, <h1> empty (to avoid reprocessing).  
    - Inputs: Rows from SEO information.  
    - Outputs: Filtered rows for processing.  
    - Edge cases: Empty or malformed fields causing filter to exclude all rows.

  - **If1**  
    - Type: If node  
    - Role: Checks if Keyword exists (non-null/non-empty) to validate filtered rows.  
    - Config: Condition to check existence of Keyword.  
    - Inputs: Filter1 output.  
    - Outputs: True branch proceeds to batch processing, False drops items.  
    - Edge cases: Keywords with invalid characters or empty strings.

  - **Loop Over Items**  
    - Type: SplitInBatches  
    - Role: Processes SEO rows one at a time (or in defined batch size) to handle multiple keywords sequentially.  
    - Config: Default batch size (usually 1).  
    - Inputs: Rows passing If1.  
    - Outputs: Single item per execution to downstream nodes.  
    - Edge cases: Large datasets may slow processing; batch size should be adjusted based on expected volume.

---

#### 1.2 Competitor Research and Analysis

- **Overview:**  
  For each keyword, this block queries Apify to retrieve the top 10 Google organic results and uses Firecrawl to scrape the first five competitor URLs to extract heading tags (H1-H4). It then processes the scraped markdown data to extract structured heading information.

- **Nodes Involved:**  
  - Apify  
  - Scrape 1, Scrape 2, Scrape 3, Scrape 4, Scrape 5  
  - Titre 1, Titre 2, Titre 3, Titre 4, Titre 5  

- **Node Details:**  

  - **Apify**  
    - Type: HTTP Request  
    - Role: Calls Apify API to search Google with the current keyword and retrieve organic results.  
    - Config: POST request with JSON body specifying countryCode: "fr", languageCode: "fr", maxPagesPerQuery: 1, resultsPerPage: 10, queries set to current keyword.  
    - Inputs: Single keyword from Loop Over Items.  
    - Outputs: JSON with organicResults array containing URLs, titles, descriptions.  
    - Credentials: Apify API key configured.  
    - Edge cases: API rate limits, timeouts, malformed responses, network errors.

  - **Scrape 1 to Scrape 5**  
    - Type: Firecrawl Scraper nodes  
    - Role: Scrape competitor URLs extracted from Apify organicResults[0..4].  
    - Config: Scrape only H1-H4 tags; continues on error to avoid workflow failure.  
    - Inputs: Each receives URL from corresponding Apify result.  
    - Outputs: Markdown content with headings for each competitor.  
    - Credentials: Firecrawl API key configured.  
    - Edge cases: Site blocking scraper, missing tags, network issues, malformed HTML.

  - **Titre 1 to Titre 5**  
    - Type: Code nodes (JavaScript)  
    - Role: Parse scraped markdown to extract and group headings by level (h1-h6).  
    - Config: Regex extracts headings, strips markdown links, groups by heading level, returns structured JSON with all heading levels.  
    - Inputs: Markdown content from corresponding Scrape node.  
    - Outputs: JSON with keys h1 to h6 and their extracted titles or placeholders if none found.  
    - Edge cases: Unexpected markdown structure, empty or missing data, regex failures.

---

#### 1.3 Meta Tags and H1 Generation

- **Overview:**  
  Using the competitor data and client information, this block generates optimized meta title, meta description, and H1 tag for the page using Claude Sonnet 4 model. The output is parsed into structured JSON format.

- **Nodes Involved:**  
  - Anthropic Chat Model2  
  - Structured Output Parser2  
  - Meta tag + h1  

- **Node Details:**  

  - **Anthropic Chat Model2**  
    - Type: Langchain Anthropic LM Chat node  
    - Role: Runs Claude AI model "claude-sonnet-4-20250514" to generate SEO meta tags based on input prompt.  
    - Config: Model selection with caching; no special options; retries handled downstream.  
    - Inputs: Prompt text from Meta tag + h1 node (agent).  
    - Outputs: AI-generated response text.  
    - Credentials: Anthropic API key configured.  
    - Edge cases: Model timeouts, API rate limits, malformed prompt inputs.

  - **Structured Output Parser2**  
    - Type: Langchain Structured Output Parser  
    - Role: Parses AI output into JSON with fields meta_title, meta_description, h1 as per example schema.  
    - Config: JSON schema example with character limits aligned to SEO best practices.  
    - Inputs: AI raw textual output from Anthropic Chat Model2.  
    - Outputs: Structured JSON object with meta tags and h1.  
    - Edge cases: Parsing errors if AI output deviates from expected format.

  - **Meta tag + h1**  
    - Type: Langchain Agent  
    - Role: Prepares prompt for Claude AI by combining main keyword, page info, client info, competitor URLs, competitor meta info, competitor headings (from Titre nodes), and description.  
    - Config: Detailed prompt with instructions to generate meta title (65 chars max), meta description (165 chars max), and h1 (70 chars max), including SEO best practices. Structured output parsing enabled.  
    - Inputs: Data from Loop Over Items, SEO information, Client Information, Apify, and Titre nodes.  
    - Outputs: Prompt text for Anthropic Chat Model2 and receives parsed JSON.  
    - Edge cases: Missing competitor data, incomplete client info, prompt length limits.

---

#### 1.4 Content Brief Creation

- **Overview:**  
  This block creates a comprehensive SEO content brief including search intent analysis, content strategy, detailed page structure (H2, H3), and media recommendations based on competitor analysis and client data using Claude AI.

- **Nodes Involved:**  
  - Anthropic Chat Model  
  - Content brief  

- **Node Details:**  

  - **Anthropic Chat Model**  
    - Type: Langchain Anthropic LM Chat node  
    - Role: Runs Claude AI model to generate SEO content brief.  
    - Config: Same model as previous block (claude-sonnet-4-20250514); max retries 5; wait 5 seconds between tries.  
    - Inputs: Prompt from Content brief node.  
    - Outputs: AI-generated text content brief.  
    - Credentials: Anthropic API key.  
    - Edge cases: Model or API errors, prompt too large.

  - **Content brief**  
    - Type: Langchain Agent  
    - Role: Prepares prompt combining client site info, main keyword, generated H1, description, awareness level, competitor heading structures (all heading levels from Titre nodes), and page type.  
    - Config: Instructions to analyze search intent percentages, content strategy, MECE page structure with H2/H3 sections, rich media suggestions, and writing detail level (1-10).  
    - Inputs: Data from Loop Over Items, Client Information, Meta tag + h1, Apify, Titre 1-5 nodes.  
    - Outputs: Prompt text for Anthropic Chat Model.  
    - Edge cases: Missing competitor heading data, incomplete client info.

---

#### 1.5 Data Integration and Sheet Updates

- **Overview:**  
  This final block consolidates all generated SEO data and updates the Google Sheets document, preserving existing data and appending new meta tags, H1, and content briefs. It also loops back to process the next batch of keywords.

- **Nodes Involved:**  
  - Code  
  - Update row in sheet  

- **Node Details:**  

  - **Code**  
    - Type: Code node (JavaScript)  
    - Role: Merges existing row data from Loop Over Items with new SEO fields: <title>, <meta-desc>, <h1>, brief (content brief text).  
    - Config: Copies all original fields and overwrites or adds the new SEO fields using outputs from Meta tag + h1 and Content brief nodes.  
    - Inputs: JSON from Loop Over Items, Meta tag + h1, Content brief.  
    - Outputs: Single merged JSON item for updating Google Sheets.  
    - Edge cases: Missing outputs from upstream nodes may cause undefined fields.

  - **Update row in sheet**  
    - Type: Google Sheets node  
    - Role: Updates the original SEO sheet row corresponding to the current keyword with new SEO content.  
    - Config: Matches rows on "mots clés" (keywords) column, auto-maps new SEO fields to columns <title>, <meta-desc>, <h1>, brief. Uses the Google Sheets URL input from chat trigger.  
    - Inputs: Merged JSON item from Code node.  
    - Outputs: Confirmation of update, triggers Loop Over Items to process next keyword.  
    - Credentials: Google Sheets OAuth2 credentials configured.  
    - Edge cases: Sheet access issues, row matching errors, API rate limits.

---

### 3. Summary Table

| Node Name               | Node Type                   | Functional Role                            | Input Node(s)                  | Output Node(s)               | Sticky Note                                                                                                    |
|-------------------------|-----------------------------|-------------------------------------------|-------------------------------|-----------------------------|---------------------------------------------------------------------------------------------------------------|
| When chat message received | Langchain Chat Trigger      | Entry webhook receiving Google Sheets URL | -                             | Client Information           | # Phase 0: Setup and Configuration <br> Instructions for spreadsheet setup and input data preparation          |
| Client Information        | Google Sheets               | Reads client business and site info        | When chat message received     | SEO information              | # Phase 0: Setup and Configuration                                                                            |
| SEO information           | Google Sheets               | Reads SEO target keywords and page info    | Client Information             | Filter1                     | # Phase 0: Setup and Configuration                                                                            |
| Filter1                   | Filter                      | Filters rows with valid Keyword and empty H1 | SEO information               | If1                         | # Phase 1: Data Input and Configuration                                                                       |
| If1                       | If                          | Validates existence of Keyword             | Filter1                       | Loop Over Items              | # Phase 1: Data Input and Configuration                                                                       |
| Loop Over Items           | SplitInBatches              | Processes keywords one by one               | If1                          | Apify                       | # Phase 1: Data Input and Configuration                                                                       |
| Apify                     | HTTP Request                | Retrieves top 10 Google organic results    | Loop Over Items               | Scrape 1                    | # Phase 2: Competitor Research and Analysis                                                                   |
| Scrape 1                  | Firecrawl Scraper           | Scrapes competitor 1 URL for headings      | Apify                        | Titre 1                     | # Phase 2: Competitor Research and Analysis                                                                   |
| Scrape 2                  | Firecrawl Scraper           | Scrapes competitor 2 URL for headings      | Titre 1                      | Titre 2                     | # Phase 2: Competitor Research and Analysis                                                                   |
| Scrape 3                  | Firecrawl Scraper           | Scrapes competitor 3 URL for headings      | Titre 2                      | Titre 3                     | # Phase 2: Competitor Research and Analysis                                                                   |
| Scrape 4                  | Firecrawl Scraper           | Scrapes competitor 4 URL for headings      | Titre 3                      | Titre 4                     | # Phase 2: Competitor Research and Analysis                                                                   |
| Scrape 5                  | Firecrawl Scraper           | Scrapes competitor 5 URL for headings      | Titre 4                      | Titre 5                     | # Phase 2: Competitor Research and Analysis                                                                   |
| Titre 1                   | Code                        | Extracts and groups headings from markdown | Scrape 1                     | Scrape 2                    | # Phase 2: Competitor Research and Analysis                                                                   |
| Titre 2                   | Code                        | Extracts and groups headings from markdown | Scrape 2                     | Scrape 3                    | # Phase 2: Competitor Research and Analysis                                                                   |
| Titre 3                   | Code                        | Extracts and groups headings from markdown | Scrape 3                     | Scrape 4                    | # Phase 2: Competitor Research and Analysis                                                                   |
| Titre 4                   | Code                        | Extracts and groups headings from markdown | Scrape 4                     | Scrape 5                    | # Phase 2: Competitor Research and Analysis                                                                   |
| Titre 5                   | Code                        | Extracts and groups headings from markdown | Scrape 5                     | Meta tag + h1               | # Phase 2: Competitor Research and Analysis                                                                   |
| Meta tag + h1             | Langchain Agent             | Prepares prompt for meta tags generation   | Titre 5, Loop Over Items, Client Information, Apify | Anthropic Chat Model2  | # Phase 3: Meta Tags and H1 Generation                                                                        |
| Anthropic Chat Model2     | Langchain Anthropic LM Chat | Generates SEO meta title, description, H1 | Meta tag + h1                | Structured Output Parser2    | # Phase 3: Meta Tags and H1 Generation                                                                        |
| Structured Output Parser2 | Langchain Output Parser     | Parses AI output into structured JSON      | Anthropic Chat Model2        | Meta tag + h1               | # Phase 3: Meta Tags and H1 Generation                                                                        |
| Content brief             | Langchain Agent             | Prepares prompt for SEO content brief      | Meta tag + h1, Loop Over Items, Client Information, Apify, Titre 1-5 | Anthropic Chat Model | # Phase 4: Content Brief Creation                                                                              |
| Anthropic Chat Model      | Langchain Anthropic LM Chat | Generates detailed SEO content brief       | Content brief                | Code                        | # Phase 4: Content Brief Creation                                                                              |
| Code                      | Code                        | Merges new SEO fields with existing row data | Content brief, Meta tag + h1, Loop Over Items | Update row in sheet       | # Phase 5: Data Integration and Sheet Updates                                                                 |
| Update row in sheet       | Google Sheets               | Updates SEO sheet with generated SEO content | Code                        | Loop Over Items             | # Phase 5: Data Integration and Sheet Updates                                                                 |
| Sticky Note* nodes        | Sticky Notes                | Documentation and phase labeling            | -                           | -                           | See detailed sticky notes below                                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a webhook trigger node:**  
   - Type: Langchain Chat Trigger  
   - Configure as public webhook with response node mode enabled.  
   - This node receives a chat message containing the Google Sheets URL.

2. **Add Google Sheets node to read "Client information":**  
   - Type: Google Sheets  
   - Operation: Read rows  
   - Sheet name: "Client information"  
   - Document ID: Extract from webhook input field `chatInput`.  
   - Credentials: Google Sheets OAuth2.

3. **Add Google Sheets node to read "SEO":**  
   - Type: Google Sheets  
   - Operation: Read rows  
   - Sheet name: "SEO"  
   - Document ID: Same as above.  
   - Credentials: Google Sheets OAuth2.

4. **Add Filter node (Filter1):**  
   - Condition: Keyword is not empty AND <h1> is empty.  
   - Input: SEO sheet rows.

5. **Add If node (If1):**  
   - Condition: Keyword exists (non-empty).  
   - Input: Filter1 output.

6. **Add SplitInBatches node (Loop Over Items):**  
   - Batch size: 1 (default).  
   - Input: True output of If1.

7. **Add HTTP Request node (Apify):**  
   - Method: POST  
   - URL: `https://api.apify.com/v2/acts/nFJndFXA5zjCTuudP/run-sync-get-dataset-items`  
   - JSON Body:  
     ```json
     {
       "countryCode": "fr",
       "forceExactMatch": false,
       "includeIcons": false,
       "includeUnfilteredResults": false,
       "languageCode": "fr",
       "maxPagesPerQuery": 1,
       "mobileResults": false,
       "queries": "{{ $json['Keyword'] }}",
       "resultsPerPage": 10,
       "saveHtml": false,
       "saveHtmlToKeyValueStore": false
     }
     ```  
   - Query Parameters: timeout=240, memory=512, maxItems=10, format=json, maxTotalChargeUsd=0.50  
   - Credentials: Apify API key.

8. **Create five Firecrawl Scraper nodes (Scrape 1 to Scrape 5):**  
   - Operation: Scrape  
   - URL: Bind each to Apify organicResults[0..4].url  
   - Scrape options: Include only tags h1, h2, h3, h4  
   - Credentials: Firecrawl API key  
   - On error: Continue execution to avoid stopping workflow.

9. **Create five Code nodes (Titre 1 to Titre 5):**  
   - JavaScript code to extract and group headings from markdown content passed from corresponding Firecrawl node.  
   - Output JSON with keys h1 to h6, each containing concatenated titles or placeholder text if none found.

10. **Add Langchain Agent node (Meta tag + h1):**  
    - Prepare a detailed prompt combining:  
      - Main keyword, page, and client info  
      - Description from Loop Over Items  
      - Competitor URLs, meta titles, meta descriptions (from Apify)  
      - Competitor headings (from Titre 1-5 nodes)  
      - Client name  
    - System message instructs generating meta title (max 65 chars), meta description (max 165 chars), and h1 (max 70 chars).  
    - Enable structured output parsing.

11. **Add Langchain Anthropic LM Chat node (Anthropic Chat Model2):**  
    - Model: "claude-sonnet-4-20250514"  
    - Input: Prompt from Meta tag + h1 node.

12. **Add Langchain Structured Output Parser node (Structured Output Parser2):**  
    - JSON schema example with meta_title, meta_description, h1.  
    - Input: Output of Anthropic Chat Model2.

13. **Link Structured Output Parser2 output back to Meta tag + h1 node** for structured result usage.

14. **Add Langchain Agent node (Content brief):**  
    - Prepare prompt with:  
      - Client site info, main keyword, generated h1, description, awareness level  
      - Competitor headings (all levels from Titre 1-5 nodes)  
      - Page type  
    - System message instructs generating detailed SEO content brief including search intent analysis, content strategy, MECE structure, media suggestions, etc.

15. **Add Langchain Anthropic LM Chat node (Anthropic Chat Model):**  
    - Same Claude Sonnet 4 model  
    - Input: Prompt from Content brief node.

16. **Add Code node (Code):**  
    - Merge original row data from Loop Over Items with outputs from Meta tag + h1 and Content brief nodes.  
    - Add/overwrite columns: <title>, <meta-desc>, <h1>, brief.

17. **Add Google Sheets node (Update row in sheet):**  
    - Operation: Update  
    - Sheet name: "FR"  
    - Document ID: Same Google Sheets URL from webhook input  
    - Match row on "mots clés" column (keyword)  
    - Auto map input data columns to sheet columns  
    - Credentials: Google Sheets OAuth2.

18. **Connect Update row in sheet back to Loop Over Items:**  
    - To continue batch processing with next keyword.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|--------------|-----------------|
| Phase 0: Setup and Configuration - Copy the template spreadsheet from https://docs.google.com/spreadsheets/d/1cRlqsueCTgfMjO7AzwBsAOzTCPBrGpHSzRg05fLDnWc and fill in Client Information and SEO sheets with relevant data. | Sticky Note "Phase 0: Setup and Configuration" |
| Phase 1: Data Input and Configuration - Input Google Sheets URL via chat, validate keywords, prepare batch processing. | Sticky Note "Phase 1: Data Input and Configuration" |
| Phase 2: Competitor Research and Analysis - Uses Apify to get top 10 results, Firecrawl to scrape headings (H1-H4) from top 5 competitors, processes markdown headings. | Sticky Note "Phase 2: Competitor Research and Analysis" |
| Phase 3: Meta Tags and H1 Generation - Uses Claude AI to generate SEO meta title, description, and H1 with competitor context and client info. | Sticky Note "Phase 3: Meta Tags and H1 Generation" |
| Phase 4: Content Brief Creation - Generates detailed SEO content brief analyzing search intent, competitor content, and suggests page structure and media. | Sticky Note "Phase 4: Content Brief Creation" |
| Phase 5: Data Integration and Sheet Updates - Updates Google Sheets with generated SEO elements, maintains batch processing loop. | Sticky Note "Phase 5: Data Integration and Sheet Updates" |
| For advanced automation solutions or workflow customization, contact Growth-AI.fr via LinkedIn profiles: https://www.linkedin.com/in/allanvaccarizi/ and https://www.linkedin.com/in/hugo-marinier-%F0%9F%A7%B2-6537b633/. | Sticky Note "Need more advanced automation solutions?" |

---

**Disclaimer:**  
The provided text is exclusively from an automated n8n workflow and complies with all content policies. All data handled is legal and public.