Fully Automate Blog Creation with GPT & Publish to WordPress via Google Sheets

https://n8nworkflows.xyz/workflows/fully-automate-blog-creation-with-gpt---publish-to-wordpress-via-google-sheets-8027


# Fully Automate Blog Creation with GPT & Publish to WordPress via Google Sheets

### 1. Workflow Overview

This n8n workflow titled **"Fully Automate Blog Creation with GPT & Publish to WordPress via Google Sheets"** focuses on comprehensive SEO keyword research and analysis to support content creation strategies. It automates fetching, processing, and aggregating keyword data from various SEO APIs (notably DataForSEO), then consolidates the results into a structured Google Sheets document for further use in blog content planning or publishing.

#### Target Use Cases:
- SEO specialists needing automated, detailed keyword research.
- Content marketers planning blog topics using data-driven insights.
- Agencies or freelancers automating SEO data collection and reporting.
- Integration with blog creation tools powered by GPT or other AI (implied by title).

#### Logical Blocks:
- **1.1 Input Reception:** Trigger and initial keyword acquisition from Google Sheets.
- **1.2 Keyword Data Retrieval:** Multiple HTTP requests to DataForSEO APIs for keyword volume, SERP data, suggestions, related keywords, subtopics, backlinks, and local finder data.
- **1.3 Data Extraction and Processing:** Code nodes parse and extract meaningful data from API responses (e.g., featured snippets, organic results, SERP features).
- **1.4 Data Aggregation and Analysis:** Nodes calculate averages, detect SERP features presence, and summarize backlink and competition metrics.
- **1.5 Output to Google Sheets:** Store all processed data into various structured sheets within a Google Sheets document for reporting and review.

---

### 2. Block-by-Block Analysis

---

#### 1.1 Input Reception

- **Overview:**  
  This block initiates the workflow manually and reads keywords from a Google Sheets document to start the research process.

- **Nodes Involved:**  
  - Manual Trigger  
  - Google Sheets (Keywords input)

- **Node Details:**

  - **Manual Trigger**  
    - Type: Trigger node for manual execution.  
    - Configuration: No parameters; user starts workflow manually.  
    - Inputs: None.  
    - Outputs: Connects downstream to Google Sheets node.  
    - Edge cases: User must manually trigger; no automation otherwise.

  - **Google Sheets (Keywords)**  
    - Type: Google Sheets node to read data.  
    - Configuration: Reads from sheet "Keywords" in a specified Google Sheets document ID.  
    - Uses OAuth2 credentials for authorization.  
    - Inputs: Trigger output.  
    - Outputs: Passes keywords JSON to multiple HTTP request nodes.  
    - Edge cases: API quota limits, invalid or missing credentials, empty sheet.

---

#### 1.2 Keyword Data Retrieval

- **Overview:**  
  This block performs multiple HTTP requests to the DataForSEO API endpoints to collect various keyword-related data: search volume, SERP results, keyword suggestions, related keywords, keyword ideas, subtopics, backlinks, local finder, and SERP screenshots.

- **Nodes Involved:**  
  - HTTP SearchVolume  
  - HTTP Google SERP  
  - HTTP Keyword Suggestions  
  - HTTP Related Keywords  
  - HTTP Keyword Ideas  
  - HTTP Subtopics  
  - HTTP Backlinks  
  - HTTP Local Finder  
  - HTTP Screenshot

- **Node Details:**

  - **HTTP SearchVolume**  
    - Type: HTTP Request to DataForSEO search volume endpoint.  
    - Configured to POST JSON body with location, language, and keywords from input.  
    - Auth: HTTP Basic Auth credentials for DataForSEO.  
    - Input: Keywords from Google Sheets.  
    - Output: Search volume data for keywords.  
    - Edge cases: API limits, invalid keys, malformed requests.

  - **HTTP Google SERP**  
    - Type: HTTP Request for organic Google SERP data.  
    - POSTs JSON specifying location, language, keyword, device, depth, and parse flag.  
    - Auth: DataForSEO credentials.  
    - Output: SERP data including organic results, snippets, PAA, etc.  
    - Edge cases: Timeout for deep results, API response inconsistencies.

  - **HTTP Keyword Suggestions**  
    - Requests keyword suggestions related to the input keyword.  
    - Auth: DataForSEO HTTP Basic.  
    - Output: Suggestions list.  
    - Edge cases: Empty suggestions, API errors.

  - **HTTP Related Keywords**  
    - Requests related keywords from DataForSEO Labs endpoint.  
    - Output: Related keywords with depth=1.  
    - Edge cases: No related keywords found, API downtime.

  - **HTTP Keyword Ideas**  
    - Requests keyword ideas limited to 100 results.  
    - Output: Seed keyword ideas and related info.  
    - Edge cases: Rate limits, large payloads.

  - **HTTP Subtopics**  
    - Requests up to 10 subtopics for the input topic.  
    - Output: Content generation subtopics.  
    - Edge cases: No subtopics returned.

  - **HTTP Backlinks**  
    - Requests backlink summary for top domain extracted later.  
    - Output: Backlinks, referring domains, spam score, country, platform.  
    - Edge cases: Missing domain, API auth failure.

  - **HTTP Local Finder**  
    - Requests local finder results for keyword.  
    - Output: Local business listings.  
    - Edge cases: No local results, malformed keyword.

  - **HTTP Screenshot**  
    - Requests screenshot for a SERP task using task ID from SERP response.  
    - Output: Screenshot URL data.  
    - Edge cases: Task ID missing, screenshot generation delay.

---

#### 1.3 Data Extraction and Processing

- **Overview:**  
  This block uses code nodes to parse complex API responses and extract specific data elements such as featured snippets, organic results, PAA questions, subtopics, related keywords, and local results. It also converts boolean flags and textual data to structured outputs.

- **Nodes Involved:**  
  - Extract Snippet1  
  - Detect SERP Features1  
  - Extract Organic  
  - Extract Top Domain  
  - Extract Local  
  - Extract Suggestions  
  - Extract Related Keywords  
  - Extract Keyword Ideas  
  - Extract Subtopics  
  - Extract Volume Trend  
  - Extract PAA/PAS

- **Node Details:**

  - **Extract Snippet1**  
    - Parses SERP JSON to find featured snippet details: title, URL, domain, snippet text truncated to 200 chars.  
    - Outputs empty fields if no snippet found.  
    - Inputs: HTTP Google SERP results.  
    - Edge cases: Missing snippet data, JSON structure changes.

  - **Detect SERP Features1**  
    - Detects presence of SERP features like featured snippet, PAA, local pack, video, image pack, top stories, knowledge panel, shopping ads.  
    - Outputs boolean flags for each feature.  
    - Inputs: HTTP Google SERP results.  
    - Edge cases: Feature naming changes, empty results.

  - **Extract Organic**  
    - Extracts organic results with rank, title, URL, domain, breadcrumb, snippet.  
    - Outputs a list of organic results.  
    - Inputs: HTTP Google SERP results.  
    - Edge cases: No organic results, missing fields.

  - **Extract Top Domain**  
    - Extracts the domain of the top-ranked organic result for backlink analysis.  
    - Inputs: HTTP Google SERP results.  
    - Edge cases: Missing organic items.

  - **Extract Local**  
    - Extracts local business name and details from local finder results.  
    - Inputs: HTTP Local Finder results.  
    - Edge cases: No local pack or map results.

  - **Extract Suggestions**  
    - Parses keyword suggestion API response to extract suggested keywords with CPC, competition, and search volume.  
    - Inputs: HTTP Keyword Suggestions results.  
    - Edge cases: Missing CPC or competition fields.

  - **Extract Related Keywords**  
    - Extracts related keywords from related keywords API response.  
    - Inputs: HTTP Related Keywords results.  
    - Edge cases: Empty related keywords list.

  - **Extract Keyword Ideas**  
    - Extracts keyword ideas from keyword ideas API response with CPC, competition, and search volume.  
    - Inputs: HTTP Keyword Ideas results.  
    - Edge cases: Missing data fields.

  - **Extract Subtopics**  
    - Extracts subtopics from content generation API.  
    - Inputs: HTTP Subtopics results.  
    - Edge cases: No subtopics returned.

  - **Extract Volume Trend**  
    - Extracts monthly search volume trend and CPC data.  
    - Inputs: HTTP SearchVolume results.  
    - Edge cases: Empty monthly data.

  - **Extract PAA/PAS**  
    - Extracts People Also Ask (PAA) questions and People Also Search (PAS) related keywords with rank.  
    - Inputs: HTTP Google SERP results.  
    - Edge cases: No PAA or PAS data.

---

#### 1.4 Data Aggregation and Analysis

- **Overview:**  
  This block calculates average search volume and CPC, determines if a featured snippet exists, compiles present SERP features, averages competition, and prepares concise information for final aggregation.

- **Nodes Involved:**  
  - average search volume and CPC  
  - Yes or No (Featured Snippet detection)  
  - Wich features exist (SERP features summary)  
  - Average competition  
  - Code (PAA question extraction)  
  - Code1 (Local results first entry extraction)  
  - Merge (combines multiple data streams)  
  - Restructure data (final data consolidation)

- **Node Details:**

  - **average search volume and CPC**  
    - Aggregates multiple monthly search volume entries to calculate average search volume and CPC.  
    - Outputs average figures and count of data points.  
    - Inputs: Extract Volume Trend node output.  
    - Edge cases: Missing CPC values, division by zero.

  - **Yes or No**  
    - Converts presence of snippet title to "Yes" or "No" string for easier visualization.  
    - Inputs: Featured Snippets node output.  
    - Edge cases: Empty snippet title field.

  - **Wich features exist**  
    - Combines individual boolean SERP feature flags into a comma-separated string listing present features or "None".  
    - Inputs: SERP Features node output.  
    - Edge cases: No features found.

  - **Average competition**  
    - Calculates average competition level from multiple keyword suggestions using numeric mapping (LOW=1, MEDIUM=2, HIGH=3).  
    - Outputs aggregated competition label.  
    - Inputs: Suggestions node output.  
    - Edge cases: Unknown competition levels, empty input.

  - **Code (PAA extraction)**  
    - Extracts only the first PAA question from all PAA results for simplified output.  
    - Inputs: PAA node output.  
    - Edge cases: No PAA questions.

  - **Code1 (Local extraction)**  
    - Extracts first local business name and details from local results.  
    - Inputs: Local Results node output.  
    - Edge cases: No local results.

  - **Merge**  
    - Merges outputs from multiple previous nodes into single JSON object.  
    - Supports 8 inputs connected here, merging all collected data points.  
    - Inputs: Outputs from average search volume, snippet detection, PAA, backlinks, competition, local info, SERP features, screenshot.  
    - Edge cases: Missing inputs causing incomplete data.

  - **Restructure data**  
    - Final restructuring to prepare a clean JSON object with all relevant data fields for output to Google Sheets.  
    - Inputs: Merge output.  
    - Edge cases: Null or missing fields handled gracefully.

---

#### 1.5 Output to Google Sheets

- **Overview:**  
  Writes the processed and aggregated SEO keyword data into several distinct sheets in a Google Sheets document for reporting, visualization, or further automated processing.

- **Nodes Involved:**  
  - Overview  
  - Search Volume Trend  
  - Featured Snippets  
  - SERP Features  
  - PAA  
  - Organic Results  
  - Local Results  
  - Suggestions  
  - Keywords Ideas  
  - Subtopics  
  - Backlinks  
  - Related Keywords

- **Node Details:**

  Each node writes to a specific Google Sheets tab with structured columns mapped from the processed data. All use the same Google Sheets OAuth2 credentials and write data in append mode.

  - **Overview**  
    - Writes aggregated summary including CPC, competition, snippet presence, SERP features, screenshot URL, backlinks, referring domains, local business names, and average search volume.  
    - Sheet: "Overview".

  - **Search Volume Trend**  
    - Writes monthly search volume and CPC trends.  
    - Sheet: "Search Volume Trend".

  - **Featured Snippets**  
    - Writes featured snippet details: title, URL, domain, snippet text.  
    - Sheet: "Featured Snippets".

  - **SERP Features**  
    - Writes boolean flags for various SERP features per keyword.  
    - Sheet: "SERP Features".

  - **PAA**  
    - Writes PAA and PAS question details with type and keyword.  
    - Sheet: "PAA".

  - **Organic Results**  
    - Writes organic result listings with rank, title, URL, domain, snippet, and breadcrumbs.  
    - Sheet: "Organic Results".

  - **Local Results**  
    - Writes local business name and details.  
    - Sheet: "Local Results".

  - **Suggestions**  
    - Writes suggested keywords with CPC, competition, and search volume.  
    - Sheet: "Suggestions".

  - **Keywords Ideas**  
    - Writes keyword ideas with CPC, competition, and search volume.  
    - Sheet: "Keyword Ideas".

  - **Subtopics**  
    - Writes subtopics with rank and keyword.  
    - Sheet: "Subtopics".

  - **Backlinks**  
    - Writes backlink summary data for top organic domain.  
    - Sheet: "Backlinks".

---

### 3. Summary Table

| Node Name              | Node Type            | Functional Role                      | Input Node(s)                    | Output Node(s)                  | Sticky Note                                           |
|------------------------|----------------------|------------------------------------|---------------------------------|--------------------------------|-------------------------------------------------------|
| Manual Trigger          | Trigger              | Workflow start manual trigger      | None                            | Google Sheets                  | ## Getting Keyword                                    |
| Google Sheets          | Google Sheets        | Reads keywords from sheet           | Manual Trigger                  | HTTP Google SERP, HTTP SearchVolume, HTTP Keyword Suggestions, HTTP Related Keywords, HTTP Keyword Ideas, HTTP Subtopics |                                                       |
| HTTP SearchVolume       | HTTP Request         | Gets keyword search volume          | Google Sheets                   | Extract Volume Trend           | # Get Avertage Search Volume And CPC                  |
| Extract Volume Trend    | Code                 | Parses monthly search volume data   | HTTP SearchVolume               | Search Volume Trend            | # Get Avertage Search Volume And CPC                  |
| Search Volume Trend     | Google Sheets        | Writes search volume trend data     | Extract Volume Trend            | average search volume and CPC  |                                                       |
| average search volume and CPC | Code           | Calculates average search volume & CPC | Search Volume Trend            | Merge                         |                                                       |
| HTTP Google SERP        | HTTP Request         | Fetches Google SERP data             | Google Sheets                  | HTTP Screenshot, HTTP Local Finder, Extract Organic, Extract Snippet1, Detect SERP Features1, Extract Top Domain, Extract PAA/PAS | # Serp Research                                       |
| HTTP Screenshot         | HTTP Request         | Fetches SERP page screenshot         | HTTP Google SERP               | Merge                         | # Get Page Screenshot                                 |
| HTTP Local Finder       | HTTP Request         | Gets local business results          | HTTP Google SERP               | Extract Local                 | # Local Results                                       |
| Extract Organic         | Code                 | Extracts organic search results      | HTTP Google SERP               | Organic Results               | # Organic Results                                    |
| Organic Results         | Google Sheets        | Writes organic results               | Extract Organic                | None                         |                                                       |
| Extract Snippet1        | Code                 | Extracts featured snippet            | HTTP Google SERP               | Featured Snippets             | # Extract Featured Snippet                           |
| Featured Snippets       | Google Sheets        | Writes featured snippet info         | Extract Snippet1              | Yes or No                    |                                                       |
| Yes or No               | Code                 | Converts snippet presence to Yes/No  | Featured Snippets             | Merge                         |                                                       |
| Detect SERP Features1   | Code                 | Detects SERP feature flags           | HTTP Google SERP               | SERP Features                |                                                       |
| SERP Features           | Google Sheets        | Writes SERP features data            | Detect SERP Features1          | Wich features exist           |                                                       |
| Wich features exist     | Code                 | Summarizes SERP features present     | SERP Features                 | Merge                         |                                                       |
| Extract Top Domain      | Code                 | Extracts top organic domain          | HTTP Google SERP               | HTTP Backlinks               | # Get Top Domain Backlinks                           |
| HTTP Backlinks          | HTTP Request         | Gets backlink summary for top domain | Extract Top Domain            | Backlinks                   |                                                       |
| Backlinks               | Google Sheets        | Writes backlink data                  | HTTP Backlinks                | Merge                         |                                                       |
| Extract PAA/PAS         | Code                 | Extracts People Also Ask/Search data | HTTP Google SERP              | PAA                         | # People Also Ask                                    |
| PAA                     | Google Sheets        | Writes PAA questions                  | Extract PAA/PAS               | Code                         |                                                       |
| Code                    | Code                 | Extracts first PAA question           | PAA                          | Merge                         |                                                       |
| Extract Local           | Code                 | Extracts local business info          | HTTP Local Finder            | Local Results               | # Local Results                                       |
| Local Results           | Google Sheets        | Writes local business data            | Extract Local                | Code1                        |                                                       |
| Code1                   | Code                 | Extracts first local business entry   | Local Results                | Merge                         |                                                       |
| HTTP Keyword Suggestions| HTTP Request         | Gets keyword suggestions              | Google Sheets                | Extract Suggestions          | # Suggested Keywords                                 |
| Extract Suggestions     | Code                 | Extracts keyword suggestions           | HTTP Keyword Suggestions     | Suggestions                 |                                                       |
| Suggestions             | Google Sheets        | Writes suggested keywords             | Extract Suggestions           | Average competition         |                                                       |
| Average competition     | Code                 | Calculates average competition level  | Suggestions                  | Merge                         |                                                       |
| HTTP Related Keywords   | HTTP Request         | Gets related keywords                  | Google Sheets                | Extract Related Keywords     | # Get Related Keywords                               |
| Extract Related Keywords| Code                 | Extracts related keywords              | HTTP Related Keywords        | Related Keywords            |                                                       |
| Related Keywords        | Google Sheets        | Writes related keywords                | Extract Related Keywords      | None                         |                                                       |
| HTTP Keyword Ideas      | HTTP Request         | Gets keyword ideas                     | Google Sheets                | Extract Keyword Ideas        | # Get Keyword Ideas                                 |
| Extract Keyword Ideas   | Code                 | Extracts keyword ideas                  | HTTP Keyword Ideas           | Keywords Ideas             |                                                       |
| Keywords Ideas          | Google Sheets        | Writes keyword ideas                   | Extract Keyword Ideas         | None                         |                                                       |
| HTTP Subtopics          | HTTP Request         | Gets subtopics                         | Google Sheets                | Extract Subtopics           | # Extract Subtopics                                 |
| Extract Subtopics       | Code                 | Extracts subtopics                      | HTTP Subtopics               | Subtopics                  |                                                       |
| Subtopics               | Google Sheets        | Writes subtopics                       | Extract Subtopics             | None                         |                                                       |
| Merge                   | Merge                | Combines multiple processed data streams | average search volume and CPC, Yes or No, Code, Backlinks, Average competition, Code1, Wich features exist, HTTP Screenshot | Restructure data       | ## Creating Overview                                |
| Restructure data        | Code                 | Final data restructuring for output  | Merge                        | Overview                    |                                                       |
| Overview                | Google Sheets        | Writes final aggregated overview      | Restructure data             | None                         | ## Creating Overview                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "Manual Trigger" node**  
   - Type: Manual Trigger  
   - No parameters needed.  
   - Position: Start node.

2. **Create "Google Sheets" node (Keywords input)**  
   - Operation: Read rows from sheet named "Keywords" in your Google Sheet document.  
   - Document ID: Your Google Sheets document ID.  
   - Sheet Name: "Keywords" (or relevant).  
   - Credentials: Configure Google Sheets OAuth2 API credentials.  
   - Connect output of Manual Trigger to this node.

3. **Create multiple HTTP Request nodes for keyword data retrieval:**

   - For each DataForSEO API endpoint below, create an HTTP Request node configured as POST with JSON body and HTTP Basic Auth credentials for DataForSEO:

     - **HTTP SearchVolume**: endpoint `/v3/keywords_data/google_ads/search_volume/live`, payload includes location, language, and keywords from Google Sheets input.

     - **HTTP Google SERP**: endpoint `/v3/serp/google/organic/live/advanced`, request includes keyword, location, device, depth, parse flag.

     - **HTTP Keyword Suggestions**: endpoint `/v3/keywords_data/google_ads/keywords_for_keywords/live`.

     - **HTTP Related Keywords**: endpoint `/v3/dataforseo_labs/google/related_keywords/live`.

     - **HTTP Keyword Ideas**: endpoint `/v3/dataforseo_labs/google/keyword_ideas/live`.

     - **HTTP Subtopics**: endpoint `/v3/content_generation/generate_sub_topics/live`.

     - **HTTP Backlinks**: endpoint `/v3/backlinks/summary/live`.

     - **HTTP Local Finder**: endpoint `/v3/serp/google/local_finder/live/advanced`.

     - **HTTP Screenshot**: endpoint `/v3/serp/screenshot`.

   - Connect the output of Google Sheets node to all these HTTP Request nodes as parallel inputs.

4. **Create Code nodes to extract and parse relevant data from API responses:**

   - **Extract Volume Trend:** Parse search volume API response for monthly data.

   - **Extract Snippet1:** Extract featured snippet data from SERP results.

   - **Detect SERP Features1:** Detect presence of SERP features from SERP results.

   - **Extract Organic:** Extract organic results details.

   - **Extract Top Domain:** Extract top domain from organic results.

   - **Extract Local:** Extract local business listings from local finder results.

   - **Extract Suggestions:** Extract keyword suggestions details.

   - **Extract Related Keywords:** Extract related keywords list.

   - **Extract Keyword Ideas:** Extract ideas from keyword ideas response.

   - **Extract Subtopics:** Extract subtopics from subtopics API.

   - **Extract PAA/PAS:** Extract PAA and PAS questions and related keywords.

5. **Create additional Google Sheets nodes to write extracted data to specific sheets:**

   - Write the outputs of extraction nodes to sheets named accordingly:  
     - "Search Volume Trend"  
     - "Featured Snippets"  
     - "SERP Features"  
     - "PAA"  
     - "Organic Results"  
     - "Local Results"  
     - "Suggestions"  
     - "Keyword Ideas"  
     - "Subtopics"  
     - "Backlinks"  
     - "Related Keywords"

6. **Create code nodes for data aggregation:**

   - **average search volume and CPC:** Calculate averages from search volume trend.

   - **Yes or No:** Convert snippet presence to string "Yes"/"No".

   - **Wich features exist:** Summarize present SERP features into a string.

   - **Average competition:** Calculate competition level average from suggestions.

   - **Code (PAA):** Extract first PAA question.

   - **Code1 (Local):** Extract first local business entry.

7. **Create a "Merge" node:**

   - Number of Inputs: 8  
   - Connect outputs of:  
     - average search volume and CPC  
     - Yes or No  
     - Code (PAA)  
     - Backlinks  
     - Average competition  
     - Code1 (Local)  
     - Wich features exist  
     - HTTP Screenshot

8. **Create a "Restructure data" code node:**

   - Consolidate merged JSON into a clean object with all relevant fields.

9. **Create "Overview" Google Sheets node:**

   - Append the restructured data into the "Overview" sheet with mapped columns.

10. **Connect all flows accordingly:**

    - Manual Trigger → Google Sheets (Keywords)  
    - Google Sheets (Keywords) → All HTTP Requests (parallel)  
    - Each HTTP Request → Corresponding extraction code node  
    - Extraction code nodes → corresponding Google Sheets output nodes  
    - Various aggregation code nodes → Merge → Restructure data → Overview Google Sheets node

11. **Credentials Setup:**

    - Create and configure Google Sheets OAuth2 credentials.  
    - Create and configure HTTP Basic Auth credentials for DataForSEO API.

12. **Defaults and Constraints:**

    - Ensure Google Sheets document and sheets exist with correct names.  
    - Handle empty or missing data gracefully in code nodes.  
    - Monitor API quotas and handle rate limits/errors.

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                                                                                     |
|----------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------|
| The workflow uses DataForSEO API for extensive keyword and SERP data; ensure valid API credentials.| https://dataforseo.com/                                                                                            |
| Google Sheets is used as the central data repository; OAuth2 credentials required for access.       | https://developers.google.com/sheets/api/guides/authorizing                                                       |
| Sticky notes in the workflow document key functional blocks such as “Get Average Search Volume and CPC” and “SERP Research”. | Visual guidance inside n8n editor                                                                                   |
| The workflow depends on JSON structure from DataForSEO; API changes may require adjustments.        | Keep API documentation at hand: https://docs.dataforseo.com/                                                       |
| Workflow is designed for US English keyword research (location_name: United States, language_code: en). | Can be adapted for other locales by changing parameters in HTTP Request nodes.                                      |
| The workflow assumes manual triggering but can be automated with scheduling or webhook triggers.    | n8n supports Cron and Webhook triggers for automation.                                                             |

---

**Disclaimer:**  
The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly available.