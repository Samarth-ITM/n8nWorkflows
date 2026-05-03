Generate SEO-Optimized Titles & Meta Descriptions with Bright Data & Gemini AI

https://n8nworkflows.xyz/workflows/generate-seo-optimized-titles---meta-descriptions-with-bright-data---gemini-ai-6115


# Generate SEO-Optimized Titles & Meta Descriptions with Bright Data & Gemini AI

### 1. Workflow Overview

This workflow automates the generation of SEO-optimized titles and meta descriptions for a list of keywords by leveraging web scraping (via Bright Data), AI processing (Google Gemini), and Google Sheets for data management. It is designed for SEO specialists and digital marketers who want to analyze top-ranking search results and produce optimized metadata that aligns with dominant search intent and user expectations.

The workflow is organized into the following logical blocks:

- **1.1 Input Reception and Keyword Retrieval**: Start manual trigger and loading keywords from Google Sheets.
- **1.2 Batch Processing Loop**: Iteratively processes each keyword individually.
- **1.3 Search Result Fetching & Data Extraction**: Uses Bright Data’s SERP API to scrape Google search results and extracts relevant fields.
- **1.4 AI-based SEO Title and Meta Description Generation**: Sends structured search data to Google Gemini AI for analysis and generation.
- **1.5 Output Parsing and Structuring**: Parses AI responses into structured data fields.
- **1.6 Results Storage and Update**: Writes the newly generated SEO titles and meta descriptions back to Google Sheets.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Keyword Retrieval

**Overview:**  
This block initiates workflow execution manually and retrieves the list of keywords (with country codes) from a Google Sheets document to process.

**Nodes Involved:**  
- When clicking ‘Execute workflow’ (Manual Trigger)  
- Get Keywords (Google Sheets)

**Node Details:**  

- **When clicking ‘Execute workflow’**  
  - Type: Manual Trigger  
  - Role: Entry point to manually start the workflow.  
  - Configuration: Default, no parameters.  
  - Input/Output: No input; outputs a trigger signal.  
  - Edge Cases: None significant; user must manually execute.

- **Get Keywords**  
  - Type: Google Sheets  
  - Role: Fetches keywords and associated country codes from a specified sheet.  
  - Configuration: Connects to Google Sheets document ID `1QU9rwawCZLiYW8nlYYRMj-9OvAUNZoe2gP49KbozQqw`, sheet `gid=0`, OAuth2 credentials used.  
  - Key Expressions: None raw; reads all rows.  
  - Input: Trigger from manual trigger node.  
  - Output: List of keyword objects with fields like `Keyword`, `country code`.  
  - Failures: Google API auth errors, quota limits, sheet not found.  
  - Notes: Users must copy and prepare the Google Sheet with keywords before running.

---

#### 2.2 Batch Processing Loop

**Overview:**  
Splits keyword list into individual items and processes each keyword sequentially.

**Nodes Involved:**  
- Loop Over Items (SplitInBatches)  
- Sticky Note5 (annotation)

**Node Details:**  

- **Loop Over Items**  
  - Type: SplitInBatches  
  - Role: Iterates over each keyword one at a time to avoid overload and manage workflow steps per keyword.  
  - Configuration: Default, batch size = 1 (implied).  
  - Input: List of keywords from Google Sheets.  
  - Output: Single keyword JSON per iteration.  
  - Edge Cases: Large keyword sets may slow processing; batch size not configurable here.  

- **Sticky Note5**  
  - Purpose: Annotation stating "We loop over each item one at a time."  
  - No functional impact.

---

#### 2.3 Search Result Fetching & Data Extraction

**Overview:**  
For each keyword, it maps keyword and country code, configures the Bright Data SERP API request, fetches Google search results JSON, and extracts relevant title/meta and "People Also Ask" questions.

**Nodes Involved:**  
- set keyword (Set)  
- Fetch Google Search Results JSON (HTTP Request)  
- Map out keyword (Set)  
- Sticky Note4, Sticky Note6, Sticky Note (annotations)  
- No Operation, do nothing1 (NoOp for error continuation)

**Node Details:**  

- **set keyword**  
  - Type: Set  
  - Role: Assigns `search_term` and `country code` variables from current batch item.  
  - Configuration: Sets `search_term` = current keyword, `country code` = current country code.  
  - Input: From Loop Over Items node.  
  - Output: JSON with mapped search term and country code.  
  - Edge Cases: Missing fields in input data.

- **Fetch Google Search Results JSON**  
  - Type: HTTP Request  
  - Role: Calls Bright Data API to scrape Google search results for the keyword.  
  - Configuration:  
    - POST to `https://api.brightdata.com/request`  
    - Body parameters include zone (`serp_api1`), constructed URL with keyword (spaces replaced by '+'), country code, format raw.  
    - Authentication via HTTP header with Bright Data credential.  
    - Query parameter `async=true` for asynchronous request.  
  - Input: From set keyword node.  
  - Output: JSON response containing organic results and People Also Ask questions.  
  - Failures: Network errors, API auth failure, rate limiting, invalid zone or parameters.  
  - On error, continues with no-op node.

- **Map out keyword**  
  - Type: Set  
  - Role: Extracts and maps specific fields from Bright Data response:  
    - `titlesDescriptions`: Array of titles and descriptions from organic results.  
    - `paaQuestions`: Extracted "People Also Ask" questions.  
    - Passes along `search_term`.  
  - Input: From Fetch Google Search Results JSON.  
  - Output: JSON prepared for AI input.  
  - Edge Cases: Missing or malformed data in response.

- **No Operation, do nothing1**  
  - Type: NoOp  
  - Role: Catch errors from HTTP Request and prevent workflow failure.  
  - Input: Error output from Fetch Google Search Results JSON.

- **Sticky Notes**  
  - Sticky Note4: Instructions to copy a Google Sheet and add keywords.  
  - Sticky Note6: Reminds to map keyword and country code, update zone name on Bright Data, and run scraper.  
  - Sticky Note: Describes analysis purpose of top 10 page titles and meta descriptions.

---

#### 2.4 AI-based SEO Title and Meta Description Generation

**Overview:**  
Sends extracted search result data and keyword to Google Gemini AI model to analyze and generate SEO-optimized title and meta description along with search intent and dominant patterns.

**Nodes Involved:**  
- Generate New title and metadescriptins (LangChain Agent)  
- Google Gemini Chat Model (AI Language Model)  
- Structured Output Parser1 (Output Parser)  
- Sticky Note (annotation near these nodes)

**Node Details:**  

- **Generate New title and metadescriptins**  
  - Type: LangChain Agent (AI agent node)  
  - Role: Formulates a complex prompt instructing the AI to analyze top-ranking titles and meta descriptions, infer SEO patterns, dominant search intent, and generate optimized metadata.  
  - Configuration:  
    - Prompt includes input keyword, top SERP titles and descriptions, and “People Also Ask” questions.  
    - Output expected in structured JSON with keys: intent, dominant_patterns (title_structure, meta_structure), optimized_title, optimized_meta, cta.  
  - Input: From Map out keyword node (with data arrays).  
  - Output: AI-generated structured response.  
  - Edge Cases: AI service unavailability, malformed response, rate limits.  
  - Credentials: Uses Google Gemini API credentials.

- **Google Gemini Chat Model**  
  - Type: AI Language Model node (LangChain integration)  
  - Role: Provides the AI language model backend for the agent.  
  - Configuration: Uses Google PaLM API credentials.  
  - Input: Connected internally to the agent node.  
  - Output: Language model response streamed to agent.

- **Structured Output Parser1**  
  - Type: LangChain Structured Output Parser  
  - Role: Validates and parses AI JSON output according to given JSON schema example.  
  - Configuration: Schema matches expected keys for intent, patterns, optimized texts, and CTA.  
  - Input: AI agent output.  
  - Output: Parsed JSON data fields.  
  - Edge Cases: Parsing failures if AI output deviates from schema.

---

#### 2.5 Output Parsing and Structuring

**Overview:**  
Extracts individual fields from AI output and prepares them for storage.

**Nodes Involved:**  
- meta structure (Set)

**Node Details:**  

- **meta structure**  
  - Type: Set  
  - Role: Maps AI output JSON fields into separate named variables: keyword, intent, dominant pattern meta and title structures, optimized title and meta, and CTA.  
  - Configuration: Assignments use expressions referencing the AI output JSON.  
  - Input: From Generate New title and metadescriptins node.  
  - Output: Clean structured data ready for appending back to Google Sheets.

---

#### 2.6 Results Storage and Update

**Overview:**  
Appends or updates the Google Sheets document with new SEO metadata for each keyword.

**Nodes Involved:**  
- Create new meta and Structure (Google Sheets)

**Node Details:**  

- **Create new meta and Structure**  
  - Type: Google Sheets  
  - Role: Appends or updates rows in the original sheet with the generated SEO data fields per keyword.  
  - Configuration:  
    - Document ID and sheet same as the initial keyword source.  
    - Matching column: Keyword, to update existing rows or append new.  
    - Columns include keyword, country code, intent, dominant pattern strings, optimized title, optimized meta, and CTA.  
    - Maps input fields automatically based on set data.  
  - Input: From meta structure node.  
  - Output: Confirmation of update.  
  - Failures: Google auth errors, concurrency issues, API limits.

---

### 3. Summary Table

| Node Name                         | Node Type                       | Functional Role                                   | Input Node(s)                  | Output Node(s)                      | Sticky Note                                                                                          |
|----------------------------------|--------------------------------|-------------------------------------------------|-------------------------------|-----------------------------------|----------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger                 | Starts the workflow manually                     | -                             | Get Keywords                      |                                                                                                    |
| Get Keywords                     | Google Sheets                  | Retrieves keyword list and country codes         | When clicking ‘Execute workflow’ | Loop Over Items                  | - Make a copy of this [G sheet](https://docs.google.com/spreadsheets/d/1QU9rwawCZLiYW8nlYYRMj-9OvAUNZoe2gP49KbozQqw/edit?usp=sharing) - Add your desired keywords |
| Loop Over Items                  | SplitInBatches                 | Iterates over each keyword individually           | Get Keywords                  | set keyword, Loop Over Items      | - We loop over each item one at a time                                                             |
| set keyword                     | Set                           | Maps keyword and country code                     | Loop Over Items               | Fetch Google Search Results JSON  | - Map keyword and country code - Update the Zone name to match your zone on Bright Data - Run the scraper |
| Fetch Google Search Results JSON | HTTP Request                  | Scrapes Google SERP data via Bright Data API     | set keyword                   | Map out keyword, No Operation     | ## Analyze title and meta description formats for top 10 pages                                     |
| No Operation, do nothing1       | NoOp                          | Catches errors from scraper and prevents failure | Fetch Google Search Results JSON (error) | -                             |                                                                                                    |
| Map out keyword                | Set                           | Extracts titles, meta descriptions, PAA questions | Fetch Google Search Results JSON | Generate New title and metadescriptins |                                                                                                    |
| Generate New title and metadescriptins | LangChain Agent (AI agent)   | AI generates optimized SEO title and meta         | Map out keyword, Google Gemini Chat Model, Structured Output Parser1 | meta structure                 |                                                                                                    |
| Google Gemini Chat Model        | AI Language Model             | Provides AI model processing                      | Internal to Agent             | Internal to Agent                 |                                                                                                    |
| Structured Output Parser1       | LangChain Structured Output Parser | Parses AI JSON output                             | AI Agent output              | Generate New title and metadescriptins |                                                                                                    |
| meta structure                 | Set                           | Maps AI output to structured fields               | Generate New title and metadescriptins | Create new meta and Structure  |                                                                                                    |
| Create new meta and Structure   | Google Sheets                  | Updates Google Sheets with new SEO data           | meta structure               | -                                 |                                                                                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node** named `When clicking ‘Execute workflow’` to start the workflow manually.

2. **Add a Google Sheets node** named `Get Keywords`:  
   - Set operation to read rows from a Google Sheet.  
   - Use OAuth2 credentials for Google Sheets API (configure accordingly).  
   - Specify the spreadsheet ID: `1QU9rwawCZLiYW8nlYYRMj-9OvAUNZoe2gP49KbozQqw`.  
   - Use sheet `gid=0` (first tab).  
   - This node fetches keywords and country codes for processing.

3. **Add a SplitInBatches node** named `Loop Over Items`:  
   - Connect input from `Get Keywords`.  
   - Set batch size to 1 to process keywords sequentially.

4. **Add a Set node** named `set keyword`:  
   - Connect input from `Loop Over Items`.  
   - Configure to set two fields:  
     - `search_term` = `{{$json.Keyword}}`  
     - `country code` = `{{$json["country code"]}}`  
   - This prepares the keyword and country code for the scraping request.

5. **Add an HTTP Request node** named `Fetch Google Search Results JSON`:  
   - Connect input from `set keyword`.  
   - Configure:  
     - Method: POST  
     - URL: `https://api.brightdata.com/request`  
     - Authentication: HTTP Header Auth with Bright Data credentials (create and assign).  
     - Query parameter: `async=true`  
     - Body parameters (JSON):  
       ```json
       {
         "parameters": [
           {"name": "zone", "value": "serp_api1"},
           {"name": "url", "value": "https://www.google.com/search?q={{ $json.search_term.replaceAll(' ', '+') }}&start=0&brd_json=1"},
           {"name": "country", "value": "{{ $json['country code'] }}"},
           {"name": "format", "value": "raw"}
         ]
       }
       ```  
     - Headers: Accept: application/json  
     - On error: Continue with error output.  
   - This node fetches the Google SERP JSON for the keyword.

6. **Add a No Operation (NoOp) node** named `No Operation, do nothing1`:  
   - Connect error output of `Fetch Google Search Results JSON` here to avoid workflow failure on scraping errors.

7. **Add a Set node** named `Map out keyword`:  
   - Connect input from the successful output of `Fetch Google Search Results JSON`.  
   - Configure assignments:  
     - `titlesDescriptions`: `{{$json.organic.map(item => ({ title: item.title, description: item.description }))}}`  
     - `paaQuestions`: `{{$json.people_also_ask.map(item => item.question)}}`  
     - `search_term`: `{{$node["set keyword"].json.search_term}}`  
   - Prepares data for AI processing.

8. **Add the LangChain Agent node** named `Generate New title and metadescriptins`:  
   - Connect input from `Map out keyword`.  
   - Use Google Gemini Chat Model as AI Language Model (create credentials with Google PaLM API and assign).  
   - Set prompt as per the original: instruct AI to analyze SERP titles and meta descriptions, infer SEO patterns, and generate optimized title and meta description with structured JSON output.  
   - Enable structured output parsing.

9. **Add the Google Gemini Chat Model node**:  
   - Connect internally to the LangChain Agent.  
   - Configure with Google PaLM API credentials.

10. **Add LangChain Structured Output Parser node** named `Structured Output Parser1`:  
    - Connect to the AI agent output.  
    - Provide JSON schema example matching keys: intent, dominant_patterns (title_structure, meta_structure), optimized_title, optimized_meta, cta.

11. **Add a Set node** named `meta structure`:  
    - Connect input from `Generate New title and metadescriptins` output.  
    - Assign these fields from AI output JSON:  
      - `Keyword` = `{{$node["Loop Over Items"].json.search_term}}`  
      - `intent` = `{{$json.output.intent}}`  
      - `dominant_patterns - meta_structure` = `{{$json.output.dominant_patterns.meta_structure}}`  
      - `dominant_patterns - title_structure` = `{{$json.output.dominant_patterns.title_structure}}`  
      - `optimized_title` = `{{$json.output.optimized_title}}`  
      - `optimized_meta` = `{{$json.output.optimized_meta}}`  
      - `cta` = `{{$json.output.cta}}`

12. **Add a Google Sheets node** named `Create new meta and Structure`:  
    - Connect input from `meta structure`.  
    - Set operation to Append or Update by matching `Keyword` column.  
    - Use the same Google Sheet ID and sheet as for input keywords.  
    - Map columns for all SEO metadata fields: intent, dominant patterns, optimized title, meta description, CTA.  
    - Use OAuth2 credentials.

13. **Connect the nodes as follows:**  
    - Manual Trigger → Get Keywords → Loop Over Items → set keyword → Fetch Google Search Results JSON → Map out keyword → Generate New title and metadescriptins → meta structure → Create new meta and Structure  
    - Fetch Google Search Results JSON (on error) → No Operation, do nothing1  
    - Loop Over Items (also loops recursively to next batch item)

14. **Test the workflow:**  
    - Add keywords and country codes to the Google Sheet.  
    - Execute manually.  
    - Monitor logs and output sheet for SEO titles and meta descriptions.

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                                                                                             |
|-------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------|
| Make a copy of this [Google Sheet](https://docs.google.com/spreadsheets/d/1QU9rwawCZLiYW8nlYYRMj-9OvAUNZoe2gP49KbozQqw/edit?usp=sharing) and add your desired keywords. | Preparation step to provide input keywords for the workflow.                                               |
| Map keyword and country code; update the Bright Data zone name to match your zone before running the scraper. | Important configuration note for accurate scraping.                                                        |
| Analyze title and meta description formats for the top 10 pages to infer SEO patterns.           | Describes the rationale behind data used by the AI model.                                                 |
| Google Gemini (PaLM) API credentials are needed for the AI language model integration.           | Credential setup requirement for AI processing.                                                           |
| Bright Data HTTP Header Auth credentials required for SERP scraping API access.                   | Credential setup requirement for scraping Google search results.                                          |

---

This documentation provides a detailed and structured explanation of the workflow "Generate SEO-Optimized Titles & Meta Descriptions with Bright Data & Gemini AI". It enables technical users and automation agents to understand, reproduce, and modify the workflow reliably.