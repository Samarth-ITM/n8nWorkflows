Find Content Gaps in Competitors' Websites with InfraNodus GraphRAG for SEO

https://n8nworkflows.xyz/workflows/find-content-gaps-in-competitors--websites-with-infranodus-graphrag-for-seo-4403


# Find Content Gaps in Competitors' Websites with InfraNodus GraphRAG for SEO

### 1. Workflow Overview

This workflow is designed to identify content gaps on competitors’ websites to support market research and SEO strategies by leveraging InfraNodus GraphRAG AI-powered content analysis. It automates the extraction, enrichment, and summarization of website content from a list of competitor URLs, storing insights back into Google Sheets and optionally compiling them into a Google Docs report.

The workflow is logically divided into two main stages:

- **Stage 1: Data Enrichment**  
  1.1 Input Reception and Batch Preparation  
  1.2 Website Content Retrieval and Cleaning  
  1.3 AI-Based Content Enhancement via InfraNodus GraphRAG

- **Stage 2: Insight Generation and Reporting**  
  2.1 Aggregation and Insight Extraction from Enriched Data  
  2.2 Content Gap and Topic Question Generation  
  2.3 Updating Google Sheets and Generating Final Reports

Additionally, the workflow includes a sub-workflow for generating a comprehensive list of competitor URLs based on a market niche, using Perplexity AI and OpenAI, populating Google Sheets for subsequent analysis.

---

### 2. Block-by-Block Analysis

#### 1. Stage 1: Data Enrichment

##### 1.1 Input Reception and Batch Preparation

- **Overview:**  
  This block receives input URLs of competitor websites from a Google Sheets document, then splits the data into manageable batches to avoid API rate limits.

- **Nodes Involved:**  
  - When clicking "Execute Workflow" (Manual Trigger)  
  - Read a Google Sheets File (Google Sheets)  
  - Split In Batches (Split In Batches)  
  - Sticky Note1, Sticky Note8, Sticky Note10 (Documentation nodes)

- **Node Details:**  
  - **When clicking "Execute Workflow"**  
    - Type: Manual Trigger  
    - Role: Initiates the workflow manually.  
    - No parameters needed.  
    - Output: Triggers the Google Sheets read.

  - **Read a Google Sheets File**  
    - Type: Google Sheets (read)  
    - Role: Reads competitor data from Google Sheets, expecting columns “Company Name,” “URL,” “Topical Summary,” “Graph Summary.”  
    - Parameters:  
      - Google Sheets document ID and sheet name specified.  
    - Inputs: Trigger from manual start.  
    - Outputs: Array of companies with URLs to analyze.  
    - Possible failures: Authorization errors, empty sheet, wrong sheet name.  
    - Credential: Google Sheets OAuth2.

  - **Split In Batches**  
    - Type: Split In Batches  
    - Role: Splits the list into batches (default size 10, configurable) to avoid API overload and rate limiting.  
    - Inputs: List of URLs from Google Sheets.  
    - Outputs: Batches of URLs to be processed sequentially.  
    - Edge Cases: If batch size is too large, may cause rate limits; if too small, workflow slows down.

  - **Sticky Notes**  
    - Provide instructions about Google Sheets data structure, batch processing to avoid rate limits, and starting the workflow.

##### 1.2 Website Content Retrieval and Cleaning

- **Overview:**  
  This block fetches the content of each competitor URL via HTTP, extracts the raw HTML, and cleans it to plain text suitable for AI analysis.

- **Nodes Involved:**  
  - HTTP Request  
  - HTML Extract  
  - Clean Content (Code node)  
  - Sticky Note2, Sticky Note3

- **Node Details:**  
  - **HTTP Request**  
    - Type: HTTP Request  
    - Role: Retrieves raw HTML content from each competitor URL.  
    - Configuration:  
      - URL dynamically set from input JSON property `URL`.  
      - Follows redirects automatically.  
      - Continue on fail enabled to prevent workflow stoppage on HTTP errors.  
    - Inputs: Batch of URLs from Split In Batches.  
    - Outputs: Raw HTML response.  
    - Failure types: Network errors, 404/500 HTTP errors.

  - **HTML Extract**  
    - Type: HTML Extract  
    - Role: Extracts the whole HTML content (selector `html`) from the HTTP response for further cleaning.  
    - Parameters: No trimming of values.  
    - Inputs: Raw HTTP response.  
    - Outputs: Extracted HTML content in JSON.  
    - Edge cases: Malformed HTML or empty response.

  - **Clean Content**  
    - Type: Code (JavaScript)  
    - Role: Cleans and normalizes the extracted HTML content by removing whitespace, newlines, and limiting content length to 10,000 characters.  
    - Key code:  
      - Removes leading/trailing whitespace, newline characters, and multiple spaces.  
      - Stores cleaned text in `content` and trimmed version in `contentShort`.  
    - Inputs: Extracted HTML content.  
    - Outputs: Cleaned text content ready for AI processing.  
    - Edge cases: Empty content, content shorter than expected.

  - **Sticky Notes**  
    - Explain HTTP request purpose and content cleaning importance.

##### 1.3 AI-Based Content Enhancement via InfraNodus GraphRAG

- **Overview:**  
  Sends cleaned content to InfraNodus GraphRAG API to generate topical summaries and graph summaries describing content gaps and keyword clusters.

- **Nodes Involved:**  
  - InfraNodus GraphRAG Content Enhancer (HTTP Request)  
  - Merge (to combine batches)  
  - Sticky Note4

- **Node Details:**  
  - **InfraNodus GraphRAG Content Enhancer**  
    - Type: HTTP Request  
    - Role: Sends content to InfraNodus API for AI-powered content summarization and gap analysis.  
    - Configuration:  
      - URL: InfraNodus API endpoint with parameters to not save data, optimize for development, exclude graph visualization but include graph summary.  
      - Method: POST  
      - Request body: includes company name, AI topics flag, cleaned content (`contentShort`), and request mode set to “summary.”  
      - Authentication: HTTP Bearer with InfraNodus API Key credential.  
    - Inputs: Cleaned content per batch item.  
    - Outputs: AI-generated summaries (`Topical Summary`, `Graph Summary`).  
    - Edge cases: API key invalid, timeout, malformed content.

  - **Merge**  
    - Type: Merge  
    - Role: Combines AI responses from all batch items back into a single array maintaining position order.  
    - Inputs: Outputs from InfraNodus requests for all batches.  
    - Outputs: Aggregated AI summaries per company.  
    - Edge cases: Data mismatch if batch sizes differ or missing data.

  - **Sticky Note4**  
    - Explains the purpose of merging AI-enhanced content back to the original data rows.

---

#### 2. Stage 2: Insight Generation and Reporting

##### 2.1 Aggregation and Insight Extraction from Enriched Data

- **Overview:**  
  Aggregates topical and graph summaries from all processed companies and generates advanced insights and question prompts via InfraNodus AI.

- **Nodes Involved:**  
  - Update Google Sheets with Content Insights (Google Sheets)  
  - Wait to avoid API overload (Wait)  
  - If Node: did we process all the data? (If)  
  - Get the content from Google Sheets (Google Sheets)  
  - Aggregate (Aggregate)  
  - InfraNodus AI Advice (HTTP Request)  
  - InfraNodus Question Generator (HTTP Request)  
  - Merge1 (Merge)  
  - Sticky Note5, Sticky Note6, Sticky Note7, Sticky Note8, Sticky Note11

- **Node Details:**  
  - **Update Google Sheets with Content Insights**  
    - Type: Google Sheets (Update)  
    - Role: Updates original Google Sheets rows with AI-generated “Topical Summary” and “Graph Summary.”  
    - Configuration: Matches rows based on the URL column and updates summary columns.  
    - Inputs: Merged AI summaries per company.  
    - Outputs: Confirmation of update.  
    - Credential: Google Sheets OAuth2.  
    - Edge cases: Sheet locked, wrong URL matching, permission errors.

  - **Wait to avoid API overload**  
    - Type: Wait  
    - Role: Pauses execution to avoid hitting API rate limits during batch processing.  
    - Parameter: Wait time in seconds (default not explicitly shown).  
    - Inputs: After sheet update.  
    - Outputs: Continues to next check.

  - **If Node: did we process all the data?**  
    - Type: If  
    - Role: Checks if all batches have been processed; if not, loops back to process next batch.  
    - Condition: Checks `noItemsLeft` context flag from Split In Batches node.  
    - Outputs:  
      - True: loops back to Split In Batches node to process next batch.  
      - False: proceeds to final data aggregation and insight generation.

  - **Get the content from Google Sheets**  
    - Type: Google Sheets (Read)  
    - Role: Reads the enriched data including AI summaries from Google Sheets for further aggregation and insight extraction.  
    - Inputs: After all batches processed.  
    - Outputs: Rows with topical and graph summaries.

  - **Aggregate**  
    - Type: Aggregate  
    - Role: Aggregates “Topical Summary” and “Graph Summary” fields from all rows into combined arrays for analysis.  
    - Inputs: Google Sheets data.  
    - Outputs: Aggregated summaries.

  - **InfraNodus AI Advice**  
    - Type: HTTP Request  
    - Role: Sends aggregated topical summary text to InfraNodus API requesting AI advice and insights about main topics and discourse gaps.  
    - Configuration:  
      - Prompt: “Tell me what are the main topics and gaps in the discourse provided.”  
      - Request mode: “response.”  
      - Auth: InfraNodus API Key.  
    - Inputs: Aggregated topical summaries.  
    - Outputs: AI advice array with text insights.

  - **InfraNodus Question Generator**  
    - Type: HTTP Request  
    - Role: Similar to AI Advice node but requests AI-generated questions highlighting content gaps and topic opportunities.  
    - Prompt: Same as AI Advice but with request mode “question.”  
    - Inputs: Aggregated topical summaries.  
    - Outputs: AI-generated questions.

  - **Merge1**  
    - Type: Merge  
    - Role: Combines AI Advice and Question Generator outputs into one data structure for later reporting.  
    - Join Mode: Keep Everything, matching on “aiAdvice.”  
    - Inputs: AI Advice and Question Generator nodes.  
    - Outputs: Combined AI insights.

  - **Sticky Notes**  
    - Provide detailed explanations about batch processing, updating sheets, enrichment, and final insight synthesis.

##### 2.2 Updating Google Sheets and Generating Final Reports

- **Overview:**  
  This block saves the generated insights back to Google Sheets and optionally appends them into a Google Docs document for reporting.

- **Nodes Involved:**  
  - Google Docs (Google Docs)  
  - Sticky Note12  
  - Merge1 (from previous point)

- **Node Details:**  
  - **Google Docs**  
    - Type: Google Docs (Update)  
    - Role: Inserts the AI-generated advice and questions into a specified Google Docs document for human-readable reporting.  
    - Parameters:  
      - Document URL specified in parameters.  
      - Inserts combined AI advice text and questions.  
    - Inputs: Merged AI insights from Merge1.  
    - Credential: Google Docs OAuth2.  
    - Edge cases: Document permissions, invalid URL.

  - **Sticky Note12**  
    - Explains saving insights into Google Docs for further use.

---

#### 3. Sub-Workflow: Generate List of Competitor URLs (Optional Seed Step)

- **Overview:**  
  This optional sub-workflow generates a list of companies relevant to a market niche via AI research, populating Google Sheets with company names, URLs, and categories, which can then be analyzed by the main workflow.

- **Nodes Involved:**  
  - On form submission (Form Trigger, disabled by default)  
  - Perplexity Research (HTTP Request)  
  - OpenAI (Langchain OpenAI node)  
  - Split Out (Split Out)  
  - Loop Over Items (Split In Batches)  
  - Google Sheets (Append to sheet)  
  - Sticky Note13

- **Node Details:**  
  - **On form submission**  
    - Type: Form Trigger (disabled)  
    - Role: Accepts user input for “Niche” and “Additional Search Instructions.”  
    - Outputs: Passes form data to Perplexity Research.  
    - Edge cases: Disabled; requires enabling for use.

  - **Perplexity Research**  
    - Type: HTTP Request  
    - Role: Queries Perplexity AI API to find links related to the specified niche, requesting a JSON list of companies with names, URLs, and categories.  
    - Payload: Includes structured prompts to act as a professional researcher.  
    - Authentication: HTTP Bearer with Perplexity API key.  
    - Outputs: Raw JSON response with company information.  
    - Failures: API limits, malformed response.

  - **OpenAI (Langchain)**  
    - Type: OpenAI Langchain node  
    - Role: Parses Perplexity response into a clean JSON list of companies.  
    - Prompts OpenAI to transform raw content into a JSON array.  
    - Outputs: Parsed company list.  
    - Authentication: OpenAI API key.

  - **Split Out**  
    - Type: Split Out  
    - Role: Splits the JSON array into individual items for batch processing.

  - **Loop Over Items**  
    - Type: Split In Batches  
    - Role: Processes items individually or in small batches.

  - **Google Sheets**  
    - Type: Google Sheets (Append)  
    - Role: Adds company data (Name, URL, Category) to Google Sheets for later use.  
    - Credential: Google Sheets OAuth2.

  - **Sticky Note13**  
    - Explains purpose of this sub-workflow to seed URLs for analysis.

---

### 3. Summary Table

| Node Name                              | Node Type                  | Functional Role                                | Input Node(s)                         | Output Node(s)                        | Sticky Note                                                                                                                                                                                                                      |
|--------------------------------------|----------------------------|------------------------------------------------|-------------------------------------|-------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| When clicking "Execute Workflow"      | Manual Trigger             | Workflow start trigger                         | -                                   | Read a Google Sheets File            | ## 0. Specify the starting point - You may connect directly to Step 8 if data is already enriched.                                                                                                                              |
| Read a Google Sheets File             | Google Sheets (Read)       | Read competitor URLs and data                  | When clicking "Execute Workflow"     | Split In Batches                     | ## 1. Specify the Google Sheet file - Must contain company names, URLs, and summary columns.                                                                                                                                   |
| Split In Batches                     | Split In Batches           | Batch processing to avoid rate limits          | Read a Google Sheets File            | HTTP Request, Merge                  | ## 2. Avoid rate limit error - Split requests into batches of 10.                                                                                                                                                              |
| HTTP Request                        | HTTP Request               | Retrieve website HTML content                   | Split In Batches                    | HTML Extract                       | ## 3. Make HTTP request and extract text content from links. Clean content to extract text, not HTML.                                                                                                                         |
| HTML Extract                       | HTML Extract              | Extract raw HTML from HTTP response             | HTTP Request                       | Clean Content                      | ## 3. See above                                                                                                                                                                                                                  |
| Clean Content                     | Code                      | Clean and normalize extracted HTML content      | HTML Extract                      | InfraNodus GraphRAG Content Enhancer | ## 3. See above                                                                                                                                                                                                                  |
| InfraNodus GraphRAG Content Enhancer | HTTP Request               | AI content enhancement and summary generation   | Clean Content                    | Merge                             | ## 4. Use InfraNodus GraphRAG Content Enhancer Tool to generate topical and graph summaries.                                                                                                                                   |
| Merge                              | Merge                     | Combine batch outputs into a single list         | InfraNodus GraphRAG Content Enhancer | Update Google Sheets with Content Insights | ## 5. Add enhanced content back to original row.                                                                                                                                                                               |
| Update Google Sheets with Content Insights | Google Sheets (Update)    | Update Google Sheets with AI summaries          | Merge                            | Wait to avoid API overload          | ## 6. Enrich original Google Sheets with keywords, summaries, and content gaps.                                                                                                                                                |
| Wait to avoid API overload          | Wait                      | Wait to prevent API rate limit                    | Update Google Sheets with Content Insights | If Node: did we process all the data? | ## 7. Batch complete? Proceed to next batch or finish.                                                                                                                                                                         |
| If Node: did we process all the data? | If                        | Check if all batches processed                    | Wait to avoid API overload          | Split In Batches (continue batch) / Get the content from Google Sheets (finish) | ## 7. See above                                                                                                                                                                                                                  |
| Get the content from Google Sheets  | Google Sheets (Read)       | Read enriched data from Google Sheets            | If Node: did we process all the data? (false branch) | Aggregate                        | ## 8. Enrich original sheets with your insights.                                                                                                                                                                               |
| Aggregate                        | Aggregate                  | Aggregate topical and graph summaries             | Get the content from Google Sheets  | InfraNodus AI Advice, InfraNodus Question Generator | ## 9. Synthesize final report based on main topics and gaps.                                                                                                                                                                   |
| InfraNodus AI Advice               | HTTP Request               | Generate AI advice on main topics and gaps       | Aggregate                        | Merge1                            | ## 9. See above                                                                                                                                                                                                                  |
| InfraNodus Question Generator     | HTTP Request               | Generate AI questions on content gaps             | Aggregate                        | Merge1                            | ## 9. See above                                                                                                                                                                                                                  |
| Merge1                            | Merge                     | Combine AI advice and question outputs            | InfraNodus AI Advice, InfraNodus Question Generator | Google Docs                      | ## 10. Save insights into Google Docs for further use.                                                                                                                                                                         |
| Google Docs                       | Google Docs                | Insert AI insights into Google Docs document      | Merge1                           | -                                 | ## 10. See above                                                                                                                                                                                                                  |
| On form submission                | Form Trigger (disabled)    | Optional trigger to generate list of companies   | -                               | Perplexity Research                | ## Generate a List of URLs to Analyze - Seed the Excel file with URLs using this workflow.                                                                                                                                     |
| Perplexity Research              | HTTP Request               | Research competitor URLs via Perplexity AI        | On form submission               | OpenAI                           | ## See above                                                                                                                                                                                                                    |
| OpenAI                          | Langchain OpenAI           | Parse Perplexity response into JSON list          | Perplexity Research              | Split Out                        | ## See above                                                                                                                                                                                                                    |
| Split Out                      | Split Out                 | Split JSON list into individual items             | OpenAI                          | Loop Over Items                  | ## See above                                                                                                                                                                                                                    |
| Loop Over Items                | Split In Batches           | Prepare individual company data for insertion     | Split Out                      | Google Sheets                   | ## See above                                                                                                                                                                                                                    |
| Google Sheets                  | Google Sheets (Append)     | Append company data to Google Sheets               | Loop Over Items                | -                               | ## See above                                                                                                                                                                                                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node** named "When clicking 'Execute Workflow'".

2. **Add a Google Sheets node** (Read operation) to read competitor data:  
   - Set Google Sheets credentials (OAuth2).  
   - Configure with the Google Sheets document ID and sheet name containing columns: Company Name, URL, Topical Summary, Graph Summary.

3. **Add a Split In Batches node** connected to the Google Sheets node output:  
   - Set batch size to 10 (default).  
   - This splits the URL list to avoid rate limits.

4. **Add an HTTP Request node** connected to Split In Batches:  
   - Set URL to `={{ $json.URL }}` (dynamic from each batch item).  
   - Enable "Follow Redirects".  
   - Enable "Continue On Fail" to handle broken links gracefully.

5. **Add an HTML Extract node** connected to HTTP Request:  
   - Extract the entire HTML content using CSS selector `html`.  
   - Disable trimming.

6. **Add a Code node ("Clean Content")** connected to HTML Extract:  
   - Use JavaScript to remove whitespace, newlines, and limit content length to 10,000 characters.  
   - Store cleaned text in `content` and shortened version in `contentShort`.

7. **Add an HTTP Request node ("InfraNodus GraphRAG Content Enhancer")** connected to Clean Content:  
   - URL: InfraNodus API endpoint `https://infranodus.com/api/v1/graphAndAdvice?doNotSave=true&optimize=develop&includeGraph=false&includeGraphSummary=true`.  
   - Method: POST.  
   - Body parameters:  
     - name: "n8n_dummy_graph"  
     - aiTopics: "true"  
     - text: `={{ $json.contentShort }}`  
     - requestMode: "summary"  
   - Authentication: HTTP Bearer with InfraNodus API Key.

8. **Add a Merge node** (mode: combine, combinationMode: mergeByPosition) to combine batch results from InfraNodus requests.

9. **Add a Google Sheets node** (Update operation) to update original Google Sheets rows:  
   - Specify document ID and sheet name.  
   - Match on URL column.  
   - Update “Topical Summary” with `={{ $json.aiAdvice[0].text }}` and “Graph Summary” with `={{ $json.graphSummary }}`.

10. **Add a Wait node ("Wait to avoid API overload")** connected after updating Google Sheets:  
    - Set wait time (e.g., 1-3 seconds) to prevent API rate limits.

11. **Add an If node ("did we process all the data?")** connected to Wait node:  
    - Condition: Check `={{ $node["Split In Batches"].context.noItemsLeft }}` is false (more batches left).  
    - True branch: Loop back to Split In Batches to process next batch.  
    - False branch: Proceed to next step.

12. **Add a Google Sheets node** (Read operation) connected to the False branch of the If node:  
    - Read the enriched Google Sheets data with updated summaries.

13. **Add an Aggregate node** connected to the Google Sheets read:  
    - Aggregate “Topical Summary” and “Graph Summary” fields into arrays.

14. **Add two HTTP Request nodes** connected to Aggregate node:  
    - "InfraNodus AI Advice":  
      - POST to InfraNodus API with parameters: aiTopics=true, prompt=“Tell me what are the main topics and gaps in the discourse provided”, requestMode=response, text=aggregated Topical Summaries.  
      - Authentication: InfraNodus API Key.

    - "InfraNodus Question Generator":  
      - Same as above, but requestMode=question.

15. **Add a Merge node (Merge1)** to combine AI Advice and Question Generator results using joinMode “keepEverything” matching on the “aiAdvice” field.

16. **Add a Google Docs node** connected to Merge1:  
    - Operation: Update.  
    - Document URL: Your Google Docs report file.  
    - Insert combined AI advice and questions into the document.  
    - Authenticate with Google Docs OAuth2.

---

**Optional Sub-Workflow: Generate Competitor URLs**

1. **Create a Form Trigger node** (disabled by default) to accept a niche and additional instructions.

2. **Add an HTTP Request node ("Perplexity Research")**:  
   - POST to Perplexity AI chat completion endpoint.  
   - Construct prompt to find links related to niche, requesting JSON list.

3. **Add an OpenAI Langchain node** to parse Perplexity response into clean JSON array.

4. **Add a Split Out node** to separate JSON array items.

5. **Add a Split In Batches node** to loop over items.

6. **Add a Google Sheets node (Append operation)** to add company Name, URL, and Category to a Google Sheets document.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                    | Context or Link                                                                                                                    |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------|
| Google Sheets template must include columns: Company Name, URL, Topical Summary, Graph Summary. Use column names exactly as in [Google Sheets Template](https://docs.google.com/spreadsheets/d/14qR7Gi8SRCd3eM6_V3ftRcDODkEFAILEqogjUvk7hKs/edit?usp=sharing) | Google Sheets template for input data                                                                                            |
| Batch requests into groups of 10 to avoid API rate limits and overloading InfraNodus or other APIs.                                                                                                                                             | Rate limit management                                                                                                            |
| InfraNodus API Key and Google API keys are required and must be properly configured in n8n credentials before running the workflow.                                                                                                            | API credentials setup                                                                                                            |
| The workflow supports an optional step to generate competitor URLs via AI research using Perplexity and OpenAI, which must be enabled separately.                                                                                              | Optional competitor URL generation                                                                                                |
| Final AI insights are saved both back to Google Sheets and optionally exported to Google Docs for reporting purposes.                                                                                                                         | Reporting and data reuse                                                                                                         |
| InfraNodus API endpoints used: `/api/v1/graphAndAdvice` with parameters to control saving, optimization mode, and inclusion of graph summaries.                                                                                                | InfraNodus API documentation                                                                                                    |
| The workflow includes multiple sticky notes providing detailed instructions and best practices at every major step.                                                                                                                           | Inline workflow documentation                                                                                                   |

---

**Disclaimer:**  
The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected material. All manipulated data are legal and publicly accessible.