Generate SEO Content with Claude AI, Competitor Analysis & Supabase RAG

https://n8nworkflows.xyz/workflows/generate-seo-content-with-claude-ai--competitor-analysis---supabase-rag-8945


# Generate SEO Content with Claude AI, Competitor Analysis & Supabase RAG

### 1. Workflow Overview

This workflow automates the generation of SEO content by integrating competitor analysis, AI-driven content creation with Claude AI (Anthropic Claude Sonnet 4), and contextual enhancement using a Supabase vector store for retrieval-augmented generation (RAG). It is designed for SEO specialists, content marketers, and digital agencies seeking to optimize website pages with data-driven and AI-augmented SEO meta tags, headers, and content briefs.

The workflow is structured into five main logical blocks:

- **1.1 Input Reception and Configuration:** Receives configuration and SEO target data via a chat trigger and Google Sheets, validating and filtering for processable keywords.
- **1.2 Competitor Research and Content Scraping:** Retrieves top organic search results via Apify API, scrapes competitor page headings using Firecrawl, and extracts hierarchical heading structures.
- **1.3 AI-Powered SEO Meta Tags and H1 Generation:** Uses Claude AI with RAG via Supabase to generate optimized meta titles, descriptions, and H1 headers based on client info and competitor data.
- **1.4 Content Brief Creation:** Builds detailed SEO content briefs with search intent analysis and structured page outlines using Claude AI and RAG.
- **1.5 Data Integration and Sheet Updates:** Consolidates all generated SEO outputs and updates the original Google Sheets rows, iterating over multiple keywords.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Configuration

**Overview:**  
This block initializes the workflow by receiving a Google Sheets URL via a chat message, then reading client-specific and SEO configuration sheets. It filters and validates keywords to prepare for batch processing.

**Nodes Involved:**  
- When chat message received  
- Client Information (Google Sheets)  
- SEO information (Google Sheets)  
- Filter1 (Filter)  
- If1 (If)  
- Loop Over Items (Split in Batches)  

**Node Details:**

- **When chat message received**  
  - *Type:* Chat Trigger (Langchain)  
  - *Role:* Entry point, receives Google Sheets URL from chat input.  
  - *Config:* Public webhook, response mode returns output node.  
  - *Edge Cases:* Invalid or missing chat input, webhook failures, malformed URLs.  
  - *Output:* Passes to Client Information.

- **Client Information**  
  - *Type:* Google Sheets (Read)  
  - *Role:* Reads client business details including client name, info, Supabase DB name, tone, restrictions.  
  - *Config:* Reads sheet "Client information" from provided Google Sheets doc ID.  
  - *Edge Cases:* Missing or malformed sheet, credential errors.  
  - *Output:* Passes to SEO information.

- **SEO information**  
  - *Type:* Google Sheets (Read)  
  - *Role:* Reads SEO target data such as page, main keyword, awareness, page type.  
  - *Config:* Reads sheet "SEO" from same document.  
  - *Edge Cases:* Missing fields, empty rows.  
  - *Output:* Feeds into Filter1.

- **Filter1**  
  - *Type:* Filter  
  - *Role:* Filters rows where 'Keyword' is not empty and '<h1>' is empty to process only new keywords needing SEO content.  
  - *Config:* Conditions: Keyword not empty AND <h1> empty.  
  - *Edge Cases:* Empty keyword rows, unexpected data types.  
  - *Input:* SEO information output.  
  - *Output:* Passes filtered items to If1.

- **If1**  
  - *Type:* If  
  - *Role:* Checks existence of Keyword to avoid processing invalid entries.  
  - *Config:* Condition: Keyword exists (non-empty).  
  - *Output:* Valid keywords proceed to Loop Over Items, invalid ones are ignored.  
  - *Edge Cases:* Empty or null keyword values.

- **Loop Over Items**  
  - *Type:* SplitInBatches  
  - *Role:* Processes keywords sequentially in batches to handle multiple rows efficiently.  
  - *Config:* Default batch size (not explicitly set here).  
  - *Output:* On batch start, triggers competitor research block.

---

#### 1.2 Competitor Research and Content Scraping

**Overview:**  
Performs competitor keyword research by querying Apify’s Google search API and scraping the top 5 competitor URLs for heading tags. Extracts detailed heading structures for SEO insights.

**Nodes Involved:**  
- Apify (HTTP Request)  
- Scrape 1 to Scrape 5 (Firecrawl Scraper)  
- Titre 1 to Titre 5 (Code nodes extracting headings)  

**Node Details:**

- **Apify**  
  - *Type:* HTTP Request  
  - *Role:* Calls Apify API to fetch top 10 organic search results for the current keyword.  
  - *Config:* POST request with JSON body specifying country=fr, language=fr, queries from Keyword, 10 results max.  
  - *Credentials:* HTTP Header Auth with Apify account.  
  - *Timeout:* 5 minutes.  
  - *Edge Cases:* API rate limits, network timeouts, invalid query.  
  - *Output:* Passes organicResults URLs to Scrape nodes.

- **Scrape 1 to Scrape 5**  
  - *Type:* Firecrawl Scraper  
  - *Role:* Scrapes competitor web pages (top 5 URLs) for h1-h4 tags to analyze page structure.  
  - *Config:* URL dynamically set from Apify organicResults[i].url; scrapes only h1-h4 tags.  
  - *Credentials:* Firecrawl API.  
  - *Error Handling:* Set to continue on error to avoid workflow stop if one scrape fails.  
  - *Output:* Passes scraped markdown content to corresponding Titre nodes.

- **Titre 1 to Titre 5**  
  - *Type:* Code  
  - *Role:* Extracts and groups markdown headings (h1 to h6) from the scraped content.  
  - *Config:* JavaScript function parsing markdown with regex to collect heading texts by level, stripping markdown links.  
  - *Output:* JSON object with keys h1..h6, each listing headings or a "no h{n}" message.  
  - *Edge Cases:* Unrecognized structure or empty markdown input logs and returns default no headings.  
  - *Input:* Scrape nodes output.  
  - *Output:* Feeds competitor heading data into AI nodes in next block.

---

#### 1.3 AI-Powered SEO Meta Tags and H1 Generation

**Overview:**  
Generates SEO-optimized meta title, meta description, and H1 header using Claude AI, enriched by competitor data and client database info fetched via Supabase vector store.

**Nodes Involved:**  
- Embeddings OpenAI2  
- Supabase Vector Store2  
- Anthropic Chat Model2  
- Structured Output Parser2  
- Meta tag + h1 (Langchain Agent)  

**Node Details:**

- **Embeddings OpenAI2**  
  - *Type:* OpenAI Embeddings  
  - *Role:* Computes embeddings for the current keyword or query to retrieve relevant documents from Supabase.  
  - *Credentials:* OpenAI API.  
  - *Output:* Embeddings passed to Supabase Vector Store2.

- **Supabase Vector Store2**  
  - *Type:* Vector Store (Supabase)  
  - *Role:* Retrieves top 5 relevant documents from client-specific Supabase DB for RAG context.  
  - *Config:* Table name and query parameters dynamically set from client info.  
  - *Credentials:* Supabase API.  
  - *Output:* Supplies contextual knowledge to Anthropic Chat Model2.

- **Anthropic Chat Model2**  
  - *Type:* Claude AI Chat Model  
  - *Role:* Generates SEO meta title, description, and H1 using prompt that integrates keyword, competitor info, and RAG data.  
  - *Model:* Claude Sonnet 4 (version 20250514).  
  - *Credentials:* Anthropic API.  
  - *Edge Cases:* API rate limits, prompt formatting errors, timeout.  
  - *Output:* Raw text response sent to Structured Output Parser2.

- **Structured Output Parser2**  
  - *Type:* Structured Output Parser (Langchain)  
  - *Role:* Parses Claude AI output into structured JSON with keys: meta_title, meta_description, h1.  
  - *Config:* JSON schema example enforces max length and format.  
  - *Output:* Parsed result passed to Meta tag + h1 node.

- **Meta tag + h1**  
  - *Type:* Langchain Agent Node  
  - *Role:* Combines all inputs and prompts Claude AI to generate final SEO meta tags and H1 per keyword considering client and competitor context.  
  - *Edge Cases:* AI generation errors, malformed inputs.  
  - *Output:* Passes generated data to Content brief node.

---

#### 1.4 Content Brief Creation

**Overview:**  
Creates a detailed SEO content brief including search intent analysis, page structure outline, and rich media suggestions based on the generated H1, competitor headings, and client info.

**Nodes Involved:**  
- Embeddings OpenAI  
- Supabase Vector Store  
- Anthropic Chat Model  
- Content brief (Langchain Agent)  

**Node Details:**

- **Embeddings OpenAI**  
  - *Type:* OpenAI Embeddings  
  - *Role:* Embeds content brief prompt for retrieval from Supabase.  
  - *Credentials:* OpenAI API.  
  - *Output:* Sent to Supabase Vector Store.

- **Supabase Vector Store**  
  - *Type:* Vector Store Supabase  
  - *Role:* Retrieves relevant client documents to enrich content brief generation.  
  - *Credentials:* Supabase API.  
  - *Output:* Feeds Anthropic Chat Model.

- **Anthropic Chat Model**  
  - *Type:* Claude AI Chat Model  
  - *Role:* Generates a comprehensive SEO content brief including search intent breakdown, content strategy, MECE page structure, media suggestions, and writing recommendations.  
  - *Model:* Claude Sonnet 4.  
  - *Credentials:* Anthropic API.  
  - *Output:* Sent as content brief to next node.

- **Content brief (Langchain Agent)**  
  - *Type:* Langchain Agent Node  
  - *Role:* Accepts inputs including client info, keywords, meta tags, competitor headings to produce structured content brief.  
  - *Edge Cases:* AI generation failures, timeout, malformed competitor data.  
  - *Output:* Passes content brief JSON to Code node.

---

#### 1.5 Data Integration and Sheet Updates

**Overview:**  
Consolidates all generated SEO content, merges with existing row data, and updates the Google Sheet accordingly. The workflow loops to process all keywords.

**Nodes Involved:**  
- Code (JavaScript)  
- Update row in sheet (Google Sheets)  

**Node Details:**

- **Code**  
  - *Type:* Code  
  - *Role:* Merges original row data with generated meta title, meta description, H1, and content brief into a single JSON object for updating.  
  - *Logic:* Copies all existing properties and overwrites relevant SEO fields.  
  - *Edge Cases:* Missing or malformed inputs, JSON parse errors.  
  - *Output:* Passes merged data to Update row in sheet.

- **Update row in sheet**  
  - *Type:* Google Sheets (Update)  
  - *Role:* Updates the original Google Sheet row with new SEO data in columns: <title>, <meta-desc>, <h1>, brief.  
  - *Config:* Uses "mots clés" column to match rows for update; updates sheet "FR" in provided document.  
  - *Credentials:* Google Sheets OAuth2.  
  - *Edge Cases:* Credential expiration, API quota limits, row not found.  
  - *Output:* Triggers Loop Over Items to continue with next keyword until all processed.

---

### 3. Summary Table

| Node Name                | Node Type                             | Functional Role                                  | Input Node(s)                    | Output Node(s)                | Sticky Note                                                                                                                                                                  |
|--------------------------|-------------------------------------|-------------------------------------------------|---------------------------------|------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| When chat message received| Langchain Chat Trigger               | Entry point; receives Google Sheets URL         | -                               | Client Information            | # Phase 1: Data Input and Configuration                                                                                                                                      |
| Client Information        | Google Sheets                       | Reads client business info sheet                 | When chat message received       | SEO information              | # Phase 1: Data Input and Configuration                                                                                                                                      |
| SEO information           | Google Sheets                       | Reads SEO target data sheet                       | Client Information               | Filter1                      | # Phase 1: Data Input and Configuration                                                                                                                                      |
| Filter1                  | Filter                             | Filters rows with non-empty Keyword & empty <h1>| SEO information                 | If1                          | # Phase 1: Data Input and Configuration                                                                                                                                      |
| If1                      | If                                | Validates keyword existence                       | Filter1                        | Loop Over Items              | # Phase 1: Data Input and Configuration                                                                                                                                      |
| Loop Over Items           | SplitInBatches                    | Batch processes keywords                          | If1                           | Apify (on first batch)       | # Phase 1: Data Input and Configuration                                                                                                                                      |
| Apify                    | HTTP Request                      | Retrieves top 10 organic Google search results  | Loop Over Items (batch start)  | Scrape 1                    | # Phase 2: Competitor Research and Analysis                                                                                                                                  |
| Scrape 1                 | Firecrawl Scraper                 | Scrapes competitor #1 page headings              | Apify                          | Titre 1                     | # Phase 2: Competitor Research and Analysis                                                                                                                                  |
| Scrape 2                 | Firecrawl Scraper                 | Scrapes competitor #2 page headings              | Titre 1                       | Titre 2                     | # Phase 2: Competitor Research and Analysis                                                                                                                                  |
| Scrape 3                 | Firecrawl Scraper                 | Scrapes competitor #3 page headings              | Titre 2                       | Titre 3                     | # Phase 2: Competitor Research and Analysis                                                                                                                                  |
| Scrape 4                 | Firecrawl Scraper                 | Scrapes competitor #4 page headings              | Titre 3                       | Titre 4                     | # Phase 2: Competitor Research and Analysis                                                                                                                                  |
| Scrape 5                 | Firecrawl Scraper                 | Scrapes competitor #5 page headings              | Titre 4                       | Titre 5                     | # Phase 2: Competitor Research and Analysis                                                                                                                                  |
| Titre 1                  | Code                             | Extracts H1-H6 headings from scraped content     | Scrape 1                      | Scrape 2                    | # Phase 2: Competitor Research and Analysis                                                                                                                                  |
| Titre 2                  | Code                             | Extracts H1-H6 headings from scraped content     | Scrape 2                      | Scrape 3                    | # Phase 2: Competitor Research and Analysis                                                                                                                                  |
| Titre 3                  | Code                             | Extracts H1-H6 headings from scraped content     | Scrape 3                      | Scrape 4                    | # Phase 2: Competitor Research and Analysis                                                                                                                                  |
| Titre 4                  | Code                             | Extracts H1-H6 headings from scraped content     | Scrape 4                      | Scrape 5                    | # Phase 2: Competitor Research and Analysis                                                                                                                                  |
| Titre 5                  | Code                             | Extracts H1-H6 headings from scraped content     | Scrape 5                      | Meta tag + h1               | # Phase 2: Competitor Research and Analysis                                                                                                                                  |
| Embeddings OpenAI2       | OpenAI Embeddings                | Creates embeddings for keyword retrieval         | Meta tag + h1 (context)        | Supabase Vector Store2       | # Phase 3: Meta Tags and H1 Generation                                                                                                                                       |
| Supabase Vector Store2   | Supabase Vector Store            | Retrieves client RAG docs to enrich AI prompt    | Embeddings OpenAI2             | Anthropic Chat Model2        | # Phase 3: Meta Tags and H1 Generation                                                                                                                                       |
| Anthropic Chat Model2    | Claude AI Chat Model             | Generates SEO meta tags and H1                    | Supabase Vector Store2          | Structured Output Parser2    | # Phase 3: Meta Tags and H1 Generation                                                                                                                                       |
| Structured Output Parser2| Structured Output Parser         | Parses AI response to structured JSON             | Anthropic Chat Model2          | Meta tag + h1               | # Phase 3: Meta Tags and H1 Generation                                                                                                                                       |
| Meta tag + h1            | Langchain Agent                 | Orchestrates meta tag generation                   | Titre 5, Structured Output Parser2 | Content brief            | # Phase 3: Meta Tags and H1 Generation                                                                                                                                       |
| Embeddings OpenAI        | OpenAI Embeddings                | Embeds brief prompt for RAG retrieval              | Content brief (input)          | Supabase Vector Store        | # Phase 4: Content Brief Creation                                                                                                                                           |
| Supabase Vector Store    | Supabase Vector Store            | Retrieves client info for content brief context    | Embeddings OpenAI              | Anthropic Chat Model         | # Phase 4: Content Brief Creation                                                                                                                                           |
| Anthropic Chat Model     | Claude AI Chat Model             | Generates detailed SEO content brief                | Supabase Vector Store          | Content brief (agent)        | # Phase 4: Content Brief Creation                                                                                                                                           |
| Content brief            | Langchain Agent                 | Produces structured content brief                   | Meta tag + h1, competitor data | Code                        | # Phase 4: Content Brief Creation                                                                                                                                           |
| Code                     | Code                             | Merges all SEO data with original row for update   | Content brief                  | Update row in sheet          | # Phase 5: Data Integration and Sheet Updates                                                                                                                               |
| Update row in sheet      | Google Sheets Update            | Updates Google Sheets row with generated SEO content| Code                          | Loop Over Items             | # Phase 5: Data Integration and Sheet Updates                                                                                                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Chat Trigger Node**  
   - Name: "When chat message received"  
   - Type: Langchain Chat Trigger  
   - Configure webhook with public access and response node enabled.  
   - This node receives the Google Sheets URL with configuration data.

2. **Add Google Sheets Node to Read Client Information**  
   - Name: "Client Information"  
   - Operation: Read Rows  
   - Sheet Name: "Client information"  
   - Document ID: Extracted dynamically from chat input (e.g., `={{ $json.chatInput }}`)  
   - Credentials: Google Sheets OAuth2  

3. **Add Google Sheets Node to Read SEO Information**  
   - Name: "SEO information"  
   - Operation: Read Rows  
   - Sheet Name: "SEO"  
   - Document ID: Same as above  
   - Credentials: Same as above  
   - Connect: "Client Information" → "SEO information"

4. **Add Filter Node to Select Keywords to Process**  
   - Name: "Filter1"  
   - Condition: Keep rows with `Keyword` not empty and `<h1>` empty  
   - Connect: "SEO information" → "Filter1"

5. **Add If Node to Validate Keyword Existence**  
   - Name: "If1"  
   - Condition: `Keyword` exists (non-empty)  
   - Connect: "Filter1" → "If1"  
   - True branch connects forward; False ignored.

6. **Add SplitInBatches Node**  
   - Name: "Loop Over Items"  
   - Default batch size or as desired  
   - Connect: "If1" true output → "Loop Over Items"

7. **Add HTTP Request Node to Call Apify API**  
   - Name: "Apify"  
   - Method: POST  
   - URL: `https://api.apify.com/v2/acts/nFJndFXA5zjCTuudP/run-sync-get-dataset-items`  
   - Body (JSON): Include parameters with `"queries": "{{ $json['Keyword'] }}"`, language and country codes set to French, max results 10, etc.  
   - Credentials: HTTP Header Auth with Apify credentials  
   - Connect: "Loop Over Items" → "Apify"

8. **Add 5 Firecrawl Scraper Nodes**  
   - Names: "Scrape 1" through "Scrape 5"  
   - Each scrapes competitor URL from Apify organicResults[i].url  
   - Scrape tags: h1, h2, h3, h4  
   - Credentials: Firecrawl API  
   - Error handling: Continue on error  
   - Connect sequentially:  
     "Apify" → "Scrape 1" → "Titre 1" → "Scrape 2" → "Titre 2" → ... → "Scrape 5" → "Titre 5"

9. **Add 5 Code Nodes to Extract Headings**  
   - Names: "Titre 1" through "Titre 5"  
   - Each parses markdown from scrape to extract h1-h6 headings using regex and returns structured JSON.  
   - Connect each Scrape node output to corresponding Titre node.

10. **Add OpenAI Embeddings Node**  
    - Name: "Embeddings OpenAI2"  
    - Credentials: OpenAI API  
    - Inputs: Data from "Loop Over Items" for embeddings  
    - Connect: Before Supabase Vector Store2

11. **Add Supabase Vector Store Node**  
    - Name: "Supabase Vector Store2"  
    - Credentials: Supabase API  
    - Configuration: Retrieve top 5 documents from client-specific DB table, dynamically set table from Client Information.  
    - Connect: "Embeddings OpenAI2" → "Supabase Vector Store2"

12. **Add Anthropic Chat Model Node**  
    - Name: "Anthropic Chat Model2"  
    - Model: "claude-sonnet-4-20250514"  
    - Credentials: Anthropic API  
    - Inputs: Incorporates client info, competitor headings, keyword, and Supabase data in prompt to generate SEO meta tags and H1.  
    - Connect: "Supabase Vector Store2" → "Anthropic Chat Model2"

13. **Add Structured Output Parser Node**  
    - Name: "Structured Output Parser2"  
    - Configured with JSON schema for meta_title, meta_description, h1 fields.  
    - Connect: "Anthropic Chat Model2" → "Structured Output Parser2"

14. **Add Langchain Agent Node for Meta Tags**  
    - Name: "Meta tag + h1"  
    - Prompt: Uses data from Titre 5, competitor info from Apify, and Client Information to request SEO meta tags and H1.  
    - Connect: Inputs from "Titre 5" and "Structured Output Parser2"  
    - Output: Connects to Content brief node.

15. **Add OpenAI Embeddings and Supabase Vector Store for Content Brief**  
    - Names: "Embeddings OpenAI", "Supabase Vector Store" (without '2')  
    - Similar config as prior embeddings/vector store but for content brief generation.  
    - Connect: "Meta tag + h1" → "Embeddings OpenAI" → "Supabase Vector Store"

16. **Add Anthropic Chat Model for Content Brief**  
    - Name: "Anthropic Chat Model"  
    - Generates detailed SEO content brief including search intent, page outlines, rich media suggestions.  
    - Connect: "Supabase Vector Store" → "Anthropic Chat Model" → "Content brief"

17. **Add Langchain Agent Node for Content Brief**  
    - Name: "Content brief"  
    - Uses client info, meta tags, competitor headings, and keyword info to create structured brief.  
    - Connect: From "Anthropic Chat Model" → "Content brief" → "Code"

18. **Add Code Node to Merge Data**  
    - Name: "Code"  
    - Logic merges original row with generated meta title, description, H1, and brief.  
    - Connect: "Content brief" → "Code"

19. **Add Google Sheets Update Node**  
    - Name: "Update row in sheet"  
    - Operation: Update row by matching 'mots clés' column  
    - Sheet: "FR"  
    - Document ID: Extracted from chat input  
    - Credentials: Google Sheets OAuth2  
    - Connect: "Code" → "Update row in sheet"

20. **Loop Back to Batch Node**  
    - Connect "Update row in sheet" → "Loop Over Items" to continue processing next batch until all keywords are done.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                  | Context or Link                                                                                              |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| Phase 0 setup instructions: Copy the template Google Sheets spreadsheet from https://docs.google.com/spreadsheets/d/1cRlqsueCTgfMjO7AzwBsAOzTCPBrGpHSzRg05fLDnWc, fill in Client Information and SEO sheets with correct business and page data to initialize workflow configuration.                                                                                          | Setup instructions for client and SEO data preparation                                                      |
| The workflow uses Claude Sonnet 4 (version 20250514) via Anthropic API for all AI content generation tasks.                                                                                                                                                                                                                                                                   | AI model choice and version                                                                                  |
| The RAG database is hosted on Supabase and must be populated with client-specific documents for enhanced personalized SEO content generation. Table naming must match client info configuration.                                                                                                                                                                                | Supabase vector store configuration                                                                          |
| Competitor scraping uses Firecrawl API with error handling to continue even if some pages fail to scrape, ensuring robust operation.                                                                                                                                                                                                                                        | Firecrawl node configuration                                                                                  |
| The Apify Google Search Actor is used to get organic search results limited to 10 per keyword with French localization.                                                                                                                                                                                                                                                      | Apify actor settings                                                                                          |
| Sticky notes provide detailed phase explanations and usage instructions within the workflow canvas.                                                                                                                                                                                                                                                                           | Internal documentation within workflow UI                                                                    |
| For custom enterprise automation solutions, contact Growth-AI.fr via LinkedIn: https://www.linkedin.com/in/allanvaccarizi/ and https://www.linkedin.com/in/hugo-marinier-%F0%9F%A7%B2-6537b633/                                                                                                                                                                                | Project credits and contact information                                                                       |

---

**Disclaimer:** The provided text is derived exclusively from an automated workflow created with n8n, compliant with current content policies. It contains no illegal, offensive, or protected material. All processed data is legal and public.