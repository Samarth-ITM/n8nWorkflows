Scrape Trustpilot Reviews with DeepSeek, Analyze Sentiment with OpenAI

https://n8nworkflows.xyz/workflows/scrape-trustpilot-reviews-with-deepseek--analyze-sentiment-with-openai-2792


# Scrape Trustpilot Reviews with DeepSeek, Analyze Sentiment with OpenAI

### 1. Workflow Overview

This workflow automates the scraping of Trustpilot reviews for a specified company, extracts detailed structured information from each review using DeepSeek, analyzes the sentiment of the review text with OpenAI, and saves the enriched data into a Google Sheet. It is designed for businesses or analysts who want to monitor customer feedback and sentiment trends efficiently.

The workflow is logically divided into four main blocks:

- **1.1 Input Reception and Parameter Setup**: Defines the target company and scraping limits.
- **1.2 Scraping and Review URL Extraction**: Fetches Trustpilot review pages and extracts individual review URLs.
- **1.3 Review Detail Extraction and Sentiment Analysis**: Retrieves each review page, extracts structured review data via DeepSeek, and analyzes sentiment using OpenAI.
- **1.4 Data Persistence**: Checks for existing reviews in Google Sheets and appends or updates records with the latest extracted and analyzed data.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Parameter Setup

**Overview:**  
This block initializes the workflow by setting key parameters such as the Trustpilot company identifier and the maximum number of review pages to scrape.

**Nodes Involved:**  
- Set Parameters  
- When clicking ‘Test workflow’ (Manual Trigger)  
- Sticky Note (instructional)

**Node Details:**

- **When clicking ‘Test workflow’**  
  - Type: Manual Trigger  
  - Role: Entry point to start the workflow manually.  
  - Configuration: No parameters; triggers downstream nodes.  
  - Inputs: None  
  - Outputs: Connects to Set Parameters node.  
  - Edge Cases: None.

- **Set Parameters**  
  - Type: Set  
  - Role: Defines workflow variables `company_id` (Trustpilot company name) and `max_page` (pagination limit).  
  - Configuration:  
    - `company_id`: Default value "COMPANY" (to be replaced by user).  
    - `max_page`: Default value 2 (can be adjusted).  
  - Inputs: From Manual Trigger  
  - Outputs: To HTTP Request node "Get reviews"  
  - Edge Cases: Missing or incorrect `company_id` will cause HTTP requests to fail or return no data.

- **Sticky Note** (positioned near Set Parameters)  
  - Content: "Change to the name of the company registered on Trustpilot and the maximum number of pages to scrape"  
  - Role: User instruction for configuration.

---

#### 1.2 Scraping and Review URL Extraction

**Overview:**  
This block performs HTTP requests to Trustpilot to fetch review pages, paginates through them, and extracts individual review URLs using HTML parsing.

**Nodes Involved:**  
- Get reviews (HTTP Request)  
- Extract (HTML Extract)  
- Split Out (Split Out)  
- Limit1 (Limit)  
- Sticky Note (instructional)

**Node Details:**

- **Get reviews**  
  - Type: HTTP Request  
  - Role: Fetches Trustpilot review pages for the specified company.  
  - Configuration:  
    - URL template: `https://it.trustpilot.com/review/{{ $json.company_id }}`  
    - Pagination: Uses page query parameter incremented by `$pageCount + 1`  
    - Pagination limit: `max_page` parameter controls max pages fetched  
    - Query parameter: `sort=recency` to get most recent reviews first  
    - Request interval: 5000 ms between requests to avoid rate limits  
  - Inputs: From Set Parameters  
  - Outputs: To Extract node  
  - Edge Cases: HTTP errors, rate limiting, invalid company_id, empty pages.

- **Extract**  
  - Type: HTML Extract  
  - Role: Parses HTML content to extract review URLs from the fetched pages.  
  - Configuration:  
    - CSS selector: `article section a`  
    - Attribute extracted: `href`  
    - Returns array of URLs under key `recensioni`  
  - Inputs: From Get reviews  
  - Outputs: To Split Out node  
  - Edge Cases: Changes in Trustpilot page structure may break extraction.

- **Split Out**  
  - Type: Split Out  
  - Role: Splits the array of review URLs into individual items for processing.  
  - Configuration: Field to split: `recensioni`  
  - Inputs: From Extract  
  - Outputs: To Limit1 node  
  - Edge Cases: Empty or malformed URL arrays.

- **Limit1**  
  - Type: Limit  
  - Role: Limits the number of review URLs processed downstream (max 3 by default).  
  - Configuration: maxItems = 3  
  - Inputs: From Split Out  
  - Outputs: To Get Google Sheets node  
  - Edge Cases: Limits processing to avoid overload; may skip some reviews if set too low.

- **Sticky Note** (near Information Extractor)  
  - Content: "Extract all information with DeepSeek (remember to change base_url with https://api.deepseek.com/v1)"  
  - Role: Instruction for DeepSeek API usage.

---

#### 1.3 Review Detail Extraction and Sentiment Analysis

**Overview:**  
For each individual review URL, this block fetches the review page, extracts structured review data using DeepSeek, analyzes the sentiment of the review text with OpenAI, and prepares data for saving.

**Nodes Involved:**  
- Get Google Sheets (Google Sheets Append/Update)  
- Get rows (Google Sheets Read)  
- If (Conditional)  
- Get Single review (HTTP Request)  
- Extract review (HTML Extract)  
- Information Extractor (DeepSeek Information Extractor)  
- DeepSeek Chat Model (DeepSeek LM Chat)  
- OpenAI Chat Model (OpenAI LM Chat)  
- Sentiment Analysis (Langchain Sentiment Analysis)  
- Update sheet (Google Sheets Append/Update)  
- Sticky Notes (instructions)

**Node Details:**

- **Get Google Sheets**  
  - Type: Google Sheets (Append or Update)  
  - Role: Saves or updates review data in Google Sheets.  
  - Configuration:  
    - Document ID: Google Sheet ID (user must configure)  
    - Sheet: `gid=0` (first sheet)  
    - Mapping: Maps review fields (`Id`, `Data`, `Nome`, `Titolo`, `Testo`, `Località`, `N. Recensioni`, `URL`, `Valutazione`, `Sentiment`)  
    - Matching column: `Id` (review URL suffix)  
  - Inputs: From Limit1 node  
  - Outputs: To Get rows node  
  - Credentials: Google Sheets OAuth2  
  - Edge Cases: Authentication errors, sheet not found, mapping mismatches.

- **Get rows**  
  - Type: Google Sheets (Read)  
  - Role: Checks if the review already exists in the sheet by matching `Id`.  
  - Configuration:  
    - Lookup column: `Id`  
    - Lookup value: Extracted review ID from URL  
  - Inputs: From Get Google Sheets  
  - Outputs: To If node  
  - Edge Cases: Empty results if review not found.

- **If**  
  - Type: Conditional  
  - Role: Determines if the review is new or already saved.  
  - Configuration: Checks if `Valutazione` field is empty (indicating new review).  
  - Inputs: From Get rows  
  - Outputs: To Get Single review if new, else ends flow for existing reviews.  
  - Edge Cases: Logic depends on correct field names and data presence.

- **Get Single review**  
  - Type: HTTP Request  
  - Role: Fetches the full HTML page of the individual review.  
  - Configuration: URL constructed as `https://it.trustpilot.com{{ review URL }}`  
  - Inputs: From If node (when review is new)  
  - Outputs: To Extract review  
  - Edge Cases: HTTP errors, page structure changes.

- **Extract review**  
  - Type: HTML Extract  
  - Role: Extracts the main review HTML block for detailed parsing.  
  - Configuration:  
    - CSS selector: `article`  
    - Returns array (usually single element) under key `recensione`  
  - Inputs: From Get Single review  
  - Outputs: To Information Extractor  
  - Edge Cases: Changes in Trustpilot HTML structure.

- **DeepSeek Chat Model**  
  - Type: Language Model Chat (DeepSeek)  
  - Role: Provides the AI model endpoint for DeepSeek extraction.  
  - Configuration:  
    - Model: `deepseek-reasoner`  
    - Base URL: `https://api.deepseek.com/v1`  
  - Credentials: DeepSeek API key  
  - Inputs: None (used as AI model provider for Information Extractor)  
  - Outputs: To Information Extractor (as AI model)  
  - Edge Cases: API key invalid, rate limits.

- **Information Extractor**  
  - Type: Langchain Information Extractor  
  - Role: Extracts structured review data from the HTML snippet using DeepSeek AI.  
  - Configuration:  
    - Input text: HTML review snippet (`{{ $json.recensione }}`)  
    - System prompt: Instructs to extract specific fields without modification  
    - Attributes extracted:  
      - `autore` (author name)  
      - `valutazione` (numeric rating 1-5)  
      - `data` (date in YYYY-MM-DD)  
      - `titolo` (review title)  
      - `testo` (full review text)  
      - `n_recensioni` (total reviews by user)  
      - `nazione` (2-letter country code)  
  - Inputs: From Extract review and DeepSeek Chat Model  
  - Outputs: To Sentiment Analysis  
  - Edge Cases: AI extraction errors, malformed HTML input.

- **OpenAI Chat Model**  
  - Type: Language Model Chat (OpenAI)  
  - Role: Provides OpenAI model for sentiment analysis.  
  - Configuration: Default OpenAI model (no special options)  
  - Credentials: OpenAI API key  
  - Inputs: None (used as AI model provider for Sentiment Analysis)  
  - Outputs: To Sentiment Analysis  
  - Edge Cases: API key invalid, rate limits.

- **Sentiment Analysis**  
  - Type: Langchain Sentiment Analysis  
  - Role: Classifies the sentiment of the extracted review text.  
  - Configuration:  
    - Categories: `Positive, Neutral, Negative`  
    - System prompt: Instructs to output JSON with category and confidence only  
    - Input text: Extracted review text (`{{ $json.output.testo }}`)  
  - Inputs: From Information Extractor and OpenAI Chat Model  
  - Outputs: To Update sheet  
  - Edge Cases: AI classification errors, ambiguous sentiment.

- **Update sheet**  
  - Type: Google Sheets (Append or Update)  
  - Role: Saves the complete review data including sentiment back to Google Sheets.  
  - Configuration:  
    - Document ID and Sheet as before  
    - Maps all extracted fields plus sentiment category  
    - Matching column: `Id`  
  - Inputs: From Sentiment Analysis and Split Out (for URL)  
  - Credentials: Google Sheets OAuth2  
  - Edge Cases: Authentication errors, data mapping issues.

- **Sticky Notes** (multiple)  
  - Near Information Extractor: "Extract all information with DeepSeek (remember to change base_url with https://api.deepseek.com/v1)"  
  - Near If node: "Check if the review has already been saved to Google Drive"  
  - Near Sentiment Analysis: "Analyze review sentiment"

---

### 3. Summary Table

| Node Name             | Node Type                          | Functional Role                                  | Input Node(s)                   | Output Node(s)                  | Sticky Note                                                                                   |
|-----------------------|----------------------------------|-------------------------------------------------|--------------------------------|--------------------------------|----------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger                   | Workflow entry point                             | None                           | Set Parameters                 |                                                                                              |
| Set Parameters        | Set                              | Defines company_id and max_page parameters       | When clicking ‘Test workflow’  | Get reviews                   | Change to the name of the company registered on Trustpilot and the maximum number of pages to scrape |
| Get reviews           | HTTP Request                     | Fetches Trustpilot review pages with pagination | Set Parameters                 | Extract                       |                                                                                              |
| Extract               | HTML Extract                    | Extracts review URLs from HTML                    | Get reviews                   | Split Out                     |                                                                                              |
| Split Out             | Split Out                       | Splits array of review URLs into individual items | Extract                       | Limit1                       |                                                                                              |
| Limit1                | Limit                           | Limits number of reviews processed                | Split Out                     | Get Google Sheets             |                                                                                              |
| Get Google Sheets     | Google Sheets (Append/Update)   | Saves or updates review data                       | Limit1                        | Get rows                     |                                                                                              |
| Get rows              | Google Sheets (Read)            | Checks if review already exists                    | Get Google Sheets             | If                           |                                                                                              |
| If                    | If                             | Conditional: process new reviews only             | Get rows                     | Get Single review             | Check if the review has already been saved to Google Drive                                   |
| Get Single review     | HTTP Request                   | Fetches individual review page                     | If                           | Extract review               |                                                                                              |
| Extract review        | HTML Extract                  | Extracts review HTML block                          | Get Single review             | Information Extractor         |                                                                                              |
| DeepSeek Chat Model   | Langchain LM Chat (DeepSeek)   | Provides DeepSeek AI model for extraction          | None                         | Information Extractor         | Extract all information with DeepSeek (remember to change base_url with https://api.deepseek.com/v1) |
| Information Extractor | Langchain Information Extractor | Extracts structured review data from HTML          | Extract review, DeepSeek Chat Model | Sentiment Analysis          |                                                                                              |
| OpenAI Chat Model     | Langchain LM Chat (OpenAI)      | Provides OpenAI model for sentiment analysis       | None                         | Sentiment Analysis            |                                                                                              |
| Sentiment Analysis    | Langchain Sentiment Analysis    | Classifies review sentiment                         | Information Extractor, OpenAI Chat Model | Update sheet               | Analyze review sentiment                                                                    |
| Update sheet          | Google Sheets (Append/Update)   | Saves enriched review data including sentiment     | Sentiment Analysis            | None                         |                                                                                              |
| Sticky Note           | Sticky Note                    | Instructional notes                                | None                         | None                         | See above notes in respective blocks                                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: `When clicking ‘Test workflow’`  
   - Purpose: Start the workflow manually.

2. **Create Set Node**  
   - Name: `Set Parameters`  
   - Add two fields:  
     - `company_id` (string), default value: `"COMPANY"` (replace with actual Trustpilot company name)  
     - `max_page` (number), default value: `2` (max pages to scrape)  
   - Connect Manual Trigger → Set Parameters.

3. **Create HTTP Request Node**  
   - Name: `Get reviews`  
   - URL: `https://it.trustpilot.com/review/{{ $json.company_id }}`  
   - Enable Pagination:  
     - Parameter: `page` = `{{$pageCount + 1}}`  
     - Max requests: `{{$json.max_page}}`  
     - Request interval: 5000 ms  
   - Query parameter: `sort=recency`  
   - Connect Set Parameters → Get reviews.

4. **Create HTML Extract Node**  
   - Name: `Extract`  
   - Operation: Extract HTML content  
   - Extraction: CSS selector `article section a` → attribute `href` → return array under key `recensioni`  
   - Connect Get reviews → Extract.

5. **Create Split Out Node**  
   - Name: `Split Out`  
   - Field to split out: `recensioni`  
   - Connect Extract → Split Out.

6. **Create Limit Node**  
   - Name: `Limit1`  
   - Max items: 3 (adjust as needed)  
   - Connect Split Out → Limit1.

7. **Create Google Sheets Node (Append or Update)**  
   - Name: `Get Google Sheets`  
   - Operation: Append or Update  
   - Document ID: Your Google Sheet ID  
   - Sheet: `gid=0`  
   - Mapping: Map review fields (`Id`, `Data`, `Nome`, `Titolo`, `Testo`, `Località`, `N. Recensioni`, `URL`, `Valutazione`, `Sentiment`)  
   - Matching column: `Id`  
   - Credentials: Google Sheets OAuth2  
   - Connect Limit1 → Get Google Sheets.

8. **Create Google Sheets Node (Read)**  
   - Name: `Get rows`  
   - Operation: Read rows  
   - Lookup column: `Id`  
   - Lookup value: `={{ $('Split Out').item.json.recensioni.replace('/reviews/','') }}`  
   - Document ID and Sheet same as above  
   - Credentials: Google Sheets OAuth2  
   - Connect Get Google Sheets → Get rows.

9. **Create If Node**  
   - Name: `If`  
   - Condition: Check if `Valutazione` field is empty for the matched row  
   - Expression: `={{ $json.Valutazione === "" }}` or equivalent loose check  
   - Connect Get rows → If.

10. **Create HTTP Request Node**  
    - Name: `Get Single review`  
    - URL: `https://it.trustpilot.com{{ $('Split Out').item.json.recensioni }}`  
    - Connect If (true branch) → Get Single review.

11. **Create HTML Extract Node**  
    - Name: `Extract review`  
    - Operation: Extract HTML content  
    - Extraction: CSS selector `article` → return array under key `recensione`  
    - Connect Get Single review → Extract review.

12. **Create Langchain LM Chat Node (DeepSeek)**  
    - Name: `DeepSeek Chat Model`  
    - Model: `deepseek-reasoner`  
    - Base URL: `https://api.deepseek.com/v1`  
    - Credentials: DeepSeek API key  
    - No input connections (used as AI model provider)  
    - Connect to Information Extractor as AI model.

13. **Create Langchain Information Extractor Node**  
    - Name: `Information Extractor`  
    - Input text: `=You need to extract the review from the following HTML:  {{ $json.recensione }}`  
    - System prompt: "You are a review expert. You need to extract only the required information and report it without changing anything. All the required information is in the text."  
    - Attributes to extract:  
      - `autore` (author name, required)  
      - `valutazione` (number 1-5, required)  
      - `data` (date YYYY-MM-DD, required)  
      - `titolo` (title, required)  
      - `testo` (text, required)  
      - `n_recensioni` (number of reviews by user, required)  
      - `nazione` (country code, 2 letters, required)  
    - Connect Extract review → Information Extractor  
    - Connect DeepSeek Chat Model → Information Extractor (AI model).

14. **Create Langchain LM Chat Node (OpenAI)**  
    - Name: `OpenAI Chat Model`  
    - Credentials: OpenAI API key  
    - Connect to Sentiment Analysis as AI model.

15. **Create Langchain Sentiment Analysis Node**  
    - Name: `Sentiment Analysis`  
    - Categories: `Positive, Neutral, Negative`  
    - System prompt: "You are highly intelligent and accurate sentiment analyzer. Analyze the sentiment of the provided text. Categorize it into one of the following: {categories}. Use the provided formatting instructions. Only output the JSON."  
    - Input text: `={{ $json.output.testo }}` (review text from Information Extractor)  
    - Connect Information Extractor → Sentiment Analysis  
    - Connect OpenAI Chat Model → Sentiment Analysis (AI model).

16. **Create Google Sheets Node (Append or Update)**  
    - Name: `Update sheet`  
    - Operation: Append or Update  
    - Document ID and Sheet same as above  
    - Mapping fields:  
      - `Id`: `={{ $('Split Out').item.json.recensioni.replace('/reviews/','') }}`  
      - `URL`: `=https://it.trustpilot.com{{ $('Split Out').item.json.recensioni }}`  
      - `Data`: `={{ $('Information Extractor').item.json.output.data }}`  
      - `Nome`: `={{ $json.output.autore }}`  
      - `Testo`: `={{ $('Information Extractor').item.json.output.testo }}`  
      - `Titolo`: `={{ $('Information Extractor').item.json.output.titolo }}`  
      - `Località`: `={{ $('Information Extractor').item.json.output.nazione }}`  
      - `Sentiment`: `={{ $json.sentimentAnalysis.category }}`  
      - `Valutazione`: `={{ $('Information Extractor').item.json.output.valutazione }}`  
      - `N. Recensioni`: `={{ $('Information Extractor').item.json.output.n_recensioni }}`  
    - Matching column: `Id`  
    - Credentials: Google Sheets OAuth2  
    - Connect Sentiment Analysis → Update sheet.

---

### 5. General Notes & Resources

| Note Content                                                                                  | Context or Link                                                                                   |
|-----------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Change the `company_id` parameter to the exact company name as registered on Trustpilot.      | Configuration instruction near Set Parameters node                                              |
| Remember to update the Google Sheet document ID and ensure the sheet has the required columns.| Google Sheets nodes configuration                                                                |
| DeepSeek API base URL must be set to `https://api.deepseek.com/v1` in the DeepSeek Chat Model.| Sticky note near Information Extractor node                                                     |
| Sentiment categories are `Positive, Neutral, Negative` and must match in Sentiment Analysis node.| Sentiment Analysis node configuration                                                           |
| Pagination limit and request interval help avoid rate limiting and overloading Trustpilot servers.| HTTP Request node configuration                                                                 |
| Google Sheets OAuth2 credentials must be properly configured to allow read/write access.      | Google Sheets nodes credential requirement                                                      |
| OpenAI and DeepSeek API keys must be valid and have sufficient quota for the workflow to run. | Credentials for OpenAI Chat Model and DeepSeek Chat Model nodes                                 |

---

This documentation provides a detailed, stepwise understanding of the workflow, enabling users and AI agents to reproduce, modify, and troubleshoot the Trustpilot review scraping and sentiment analysis automation effectively.