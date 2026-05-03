Scrape & Summarize Google Maps Businesses with APIFY + GPT-4O to Sheets

https://n8nworkflows.xyz/workflows/scrape---summarize-google-maps-businesses-with-apify---gpt-4o-to-sheets-10120


# Scrape & Summarize Google Maps Businesses with APIFY + GPT-4O to Sheets

### 1. Workflow Overview

This n8n workflow automates the process of scraping local business listings from Google Maps, summarizing the extracted data into human-readable company profiles using OpenAI’s GPT-4o model, and storing the summarized results in a Google Sheet for easy access and further use. The workflow is designed for lead generation, prospecting, or business data enrichment, targeting French businesses categorized as “restaurant” in this example.

**Logical blocks:**

- **1.1 Trigger and Scraping:** Manual start node triggers the scraping of Google Maps business data via Apify’s Google Maps Scraper actor.
- **1.2 Dataset Retrieval and Deduplication:** Retrieves the scraped dataset from Apify, then removes duplicate business entries based on the company title.
- **1.3 Iterative Processing and AI Summarization:** Loops over each unique business entry to generate a concise, professional summary paragraph using OpenAI’s GPT-4o.
- **1.4 Data Storage and Rate Limiting:** Appends each summarized business record into a structured Google Sheet and implements a pause to manage rate limits before processing the next item.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger and Scraping Block

**Overview:**  
Starts the workflow manually and launches an Apify actor to scrape Google Maps for businesses matching specified criteria.

**Nodes Involved:**  
- When clicking ‘Execute workflow’ (Manual Trigger)  
- Run Google Maps Scraper (Apify)

**Node Details:**

- **When clicking ‘Execute workflow’**  
  - Type: Manual Trigger  
  - Role: Entry point to start the workflow manually for testing or batch execution.  
  - Configuration: Default, no parameters.  
  - Connections: Outputs to Run Google Maps Scraper.  
  - Edge cases: None specific; user-initiated trigger.

- **Run Google Maps Scraper**  
  - Type: Apify Node  
  - Role: Runs a specified Apify actor (“nwua9Gu5YrADL7ZDj”) to scrape Google Maps data.  
  - Configuration:  
    - Country: France (fr)  
    - Language: French  
    - Search query: “restaurant”  
    - Max places per search: 5  
    - Scrape contacts enabled  
    - Skips closed places  
    - No images or directories scraped  
    - Scrapes personal data from reviews  
  - Credentials: Uses stored Apify API credentials.  
  - Connections: Output goes to Get dataset items node.  
  - Edge cases: Possible API rate limits, actor errors, or no data returned. Must handle empty datasets gracefully.

---

#### 2.2 Dataset Retrieval and Deduplication Block

**Overview:**  
Fetches the dataset from Apify after scraping completes and removes duplicate business entries by comparing titles.

**Nodes Involved:**  
- Get dataset items (Apify)  
- Remove Duplicates

**Node Details:**

- **Get dataset items**  
  - Type: Apify Node  
  - Role: Retrieves the dataset results from the Apify Google Maps Scraper actor using a dynamic Dataset ID from prior output.  
  - Configuration:  
    - Dataset resource used with dynamic datasetId expression `={{ $json.defaultDatasetId }}`  
    - No limit or offset specified (fetches all available)  
  - Credentials: Apify API credentials reused.  
  - Connections: Outputs to Remove Duplicates.  
  - Edge cases: Dataset ID missing or invalid, network/API errors, empty dataset.

- **Remove Duplicates**  
  - Type: Remove Duplicates Node  
  - Role: Filters out duplicate entries based on the “title” field to ensure unique businesses only.  
  - Configuration:  
    - Compare mode: Selected fields  
    - Fields to compare: title  
  - Connections: Outputs to Loop Over Items node.  
  - Edge cases: Variations in business titles with minor differences might not be caught; may need normalization.

---

#### 2.3 Iterative Processing and AI Summarization Block

**Overview:**  
Iterates over each unique business entry and generates a professional summary paragraph with key business info using OpenAI GPT-4o.

**Nodes Involved:**  
- Loop Over Items (Split in Batches)  
- company summary Info (OpenAI Chat Completion)

**Node Details:**

- **Loop Over Items**  
  - Type: Split In Batches  
  - Role: Processes one business item at a time to avoid API rate limits and enable sequential AI summarization.  
  - Configuration: Default batch size 1 (implicit)  
  - Connections: Main output to company summary Info; second output used for iteration control (loops back after Pause node).  
  - Edge cases: If batch size is too large, risk hitting API limits or memory issues.

- **company summary Info**  
  - Type: OpenAI Chat Completion (LangChain)  
  - Role: Generates a concise, natural language summary paragraph for each business item using GPT-4o.  
  - Configuration:  
    - Model: gpt-4o  
    - System prompt: Describes assistant role to generate professional summaries with company name, category, address, phone, and Google Maps URL.  
    - User prompt: Uses n8n expressions to inject raw business data fields dynamically (`$json.title`, `$json.categoryName`, etc.) into the prompt.  
    - Output: JSON format with key `companySummary` containing the summary paragraph.  
  - Credentials: OpenAI API key configured.  
  - Connections: Outputs to Google maps database (Google Sheets node).  
  - Edge cases: API errors, timeout, prompt injection risks, missing data fields causing incomplete summaries.

---

#### 2.4 Data Storage and Rate Limiting Block

**Overview:**  
Appends each summarized business record to a Google Sheet and pauses briefly to prevent rate limiting or API throttling.

**Nodes Involved:**  
- Google maps database (Google Sheets)  
- Pause for rate limit (Wait)  

**Node Details:**

- **Google maps database**  
  - Type: Google Sheets Node  
  - Role: Appends processed business data along with the AI-generated summary to a predefined Google Sheet.  
  - Configuration:  
    - Document ID and sheet name set to target Google Sheet.  
    - Columns mapped explicitly with expressions to pull fields from the current loop item and AI summary (`companySummaryInfo`).  
    - Append operation used to add rows sequentially.  
  - Credentials: Google Sheets OAuth2 credentials configured.  
  - Connections: Outputs to Pause for rate limit.  
  - Edge cases: Sheets API quota exceeded, invalid document ID, permission errors.

- **Pause for rate limit**  
  - Type: Wait Node  
  - Role: Introduces a 2-second delay between processing each item to reduce risk of API rate limits and ensure stable execution.  
  - Configuration: Wait time set to 2 seconds.  
  - Connections: Loops back to Loop Over Items node to process next business.  
  - Edge cases: Long workflows may take time due to cumulative delays; timing can be adjusted as needed.

---

### 3. Summary Table

| Node Name                 | Node Type                     | Functional Role                              | Input Node(s)                | Output Node(s)             | Sticky Note                                                                                                           |
|---------------------------|-------------------------------|----------------------------------------------|-----------------------------|----------------------------|-----------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger               | Manual start of the workflow                  |                             | Run Google Maps Scraper     |                                                                                                                       |
| Run Google Maps Scraper    | Apify                         | Runs Google Maps scraping actor               | When clicking ‘Execute workflow’ | Get dataset items          | ## Run Google Maps Scraper & remove duplicates                                                                        |
| Get dataset items         | Apify                         | Retrieves scraped dataset                      | Run Google Maps Scraper      | Remove Duplicates           |                                                                                                                       |
| Remove Duplicates          | Remove Duplicates              | Removes duplicate businesses by title         | Get dataset items            | Loop Over Items             |                                                                                                                       |
| Loop Over Items            | Split In Batches              | Processes each business individually          | Remove Duplicates            | company summary Info, Loop Over Items (iteration) |                                                                                                                       |
| company summary Info       | OpenAI Chat Completion (LangChain) | Generates company summary paragraphs           | Loop Over Items              | Google maps database        | ## Company summary info generator                                                                                     |
| Google maps database       | Google Sheets                 | Appends summarized data to Google Sheet        | company summary Info         | Pause for rate limit        | ## Data Base                                                                                                           |
| Pause for rate limit       | Wait                         | Adds delay to avoid rate limiting               | Google maps database         | Loop Over Items             |                                                                                                                       |
| Sticky Note                | Sticky Note                  | Notes on scraping and deduplication block      |                             |                            | ## Run Google Maps Scraper & remove duplicates                                                                        |
| Sticky Note1               | Sticky Note                  | Notes on company summary info generation        |                             |                            | ## Company summary info generator                                                                                      |
| Sticky Note2               | Sticky Note                  | Notes on data storage in Google Sheets          |                             |                            | ## Data Base                                                                                                           |
| Sticky Note3               | Sticky Note                  | Overall workflow description and use cases     |                             |                            | ## This automation scrapes businesses from Google Maps, summarizes their information using ChatGPT, and stores results in Google Sheets. This workflow automates the entire process of lead generation from Google Maps — from scraping business info, cleaning duplicates, summarizing key details, and storing it in Google Sheets — so you can save hours of manual research and focus on outreach and closing deals. |
| Sticky Note4               | Sticky Note                  | Detailed instructions, use cases, and tips      |                             |                            | ## This workflow automates the process of scraping local business listings from Google Maps and generating clean, AI-powered summaries for each one — using Apify (community node) and OpenAI’s GPT-4o. All results are saved into Google Sheets, ready for lead generation. Setup guides and contact links included.                  |
| Sticky Note7               | Sticky Note                  | Apify manual config screenshot                  |                             |                            | ## Apify manual config ![](https://res.cloudinary.com/dya4plw3w/image/upload/v1761304716/Capture_d_e%CC%81cran_2025-10-24_a%CC%80_13.03.03_pozkdi.png) |
| Sticky Note8               | Sticky Note                  | Apify JSON config screenshot                     |                             |                            | ## Apify JSON config ![](https://res.cloudinary.com/dya4plw3w/image/upload/v1761304716/Capture_d_e%CC%81cran_2025-10-24_a%CC%80_13.03.16_m8wewv.png)  |
| Sticky Note9               | Sticky Note                  | Google Sheets column setup screenshot            |                             |                            | ## Google sheet column setups ![](https://res.cloudinary.com/dya4plw3w/image/upload/v1761305547/Capture_d_e%CC%81cran_2025-10-24_a%CC%80_13.31.49_hn4ibf.png) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node:**  
   - Add a Manual Trigger node named “When clicking ‘Execute workflow’” to start the process manually.

2. **Add Apify Node to Run Google Maps Scraper:**  
   - Add an Apify node named “Run Google Maps Scraper.”  
   - Configure with your Apify credentials.  
   - Set Actor ID to the Google Maps Scraper actor (example: “nwua9Gu5YrADL7ZDj”).  
   - In the “Custom Body” JSON, configure parameters such as:  
     - `countryCode`: “fr”  
     - `language`: “fr”  
     - `locationQuery`: “France”  
     - `searchStringsArray`: [“restaurant”]  
     - `maxCrawledPlacesPerSearch`: 5  
     - Enable `scrapeContacts` and set other scraping options as needed.  
   - Connect output from Manual Trigger to this node.

3. **Add Apify Node to Get Dataset Items:**  
   - Add another Apify node named “Get dataset items.”  
   - Use the “Datasets” resource with dynamic Dataset ID from the previous node’s output (`={{ $json.defaultDatasetId }}`).  
   - Use the same Apify credentials.  
   - Connect output from “Run Google Maps Scraper” to this node.

4. **Add Remove Duplicates Node:**  
   - Add a “Remove Duplicates” node named “Remove Duplicates.”  
   - Set comparison mode to “selectedFields.”  
   - Choose “title” as the field to compare for duplicates.  
   - Connect output from “Get dataset items” to this node.

5. **Add Split In Batches Node:**  
   - Add “Split In Batches” node named “Loop Over Items.”  
   - Default batch size 1 (process one item at a time).  
   - Connect output from “Remove Duplicates” to this node.

6. **Add OpenAI Chat Completion Node:**  
   - Add “OpenAI Chat Completion” node named “company summary Info.”  
   - Configure with OpenAI credentials (API key).  
   - Select model “gpt-4o.”  
   - Set system prompt: instruct assistant to generate professional company summaries.  
   - Set user prompt using expressions to pass current item’s fields: company name, category, address, city, country, phone, and Google Maps URL.  
   - Enable JSON output with key `companySummary`.  
   - Connect main output from “Loop Over Items” to this node.

7. **Add Google Sheets Node:**  
   - Add “Google Sheets” node named “Google maps database.”  
   - Connect to your Google Sheets account with OAuth2.  
   - Configure to append rows to the desired Google Sheet and tab (specify Document ID and sheet name).  
   - Map columns using expressions from the current item and the AI-generated summary (`companySummaryInfo` from the OpenAI node).  
   - Connect output from “company summary Info” to this node.

8. **Add Wait Node for Rate Limiting:**  
   - Add “Wait” node named “Pause for rate limit.”  
   - Set wait time to 2 seconds (adjustable).  
   - Connect output from “Google maps database” to this node.  
   - Connect output from this node back to the second output (iteration) of “Loop Over Items” node to continue processing remaining items.

9. **Verify Connections and Test:**  
   - Ensure all nodes are connected as per the workflow logic.  
   - Test by executing manually and monitor logs for errors or API issues.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           | Context or Link                                                                                          |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| This workflow automates scraping, summarizing, and storing local business data from Google Maps using Apify and OpenAI. It is designed to save hours of manual research and is ideal for sales teams, lead generators, marketers, and automation enthusiasts.                                                                                                                                                                                                                                                                                                                                                                                     | General workflow purpose and use cases.                                                                 |
| Setup requires: Apify Google Maps Scraper actor, OpenAI API key (GPT-4o recommended), and Google Sheets account for data storage.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    | Setup prerequisites.                                                                                      |
| Contact for consulting and support: [LinkedIn](https://www.linkedin.com/in/jaures-nya-83a033270/), [YouTube](https://www.youtube.com/@jauresnya), [Skool](https://www.skool.com/gaia-4903/about?ref=e0430e4c35b645ac8976b952768e9d55)                                                                                                                                                                                                                                                                                                                                                                       | Support channels and community links.                                                                    |
| Apify manual config screenshot: ![](https://res.cloudinary.com/dya4plw3w/image/upload/v1761304716/Capture_d_e%CC%81cran_2025-10-24_a%CC%80_13.03.03_pozkdi.png)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           | Visual reference for Apify actor configuration.                                                         |
| Apify JSON config screenshot: ![](https://res.cloudinary.com/dya4plw3w/image/upload/v1761304716/Capture_d_e%CC%81cran_2025-10-24_a%CC%80_13.03.16_m8wewv.png)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              | JSON configuration example for Apify input.                                                             |
| Google Sheets column setup screenshot: ![](https://res.cloudinary.com/dya4plw3w/image/upload/v1761305547/Capture_d_e%CC%81cran_2025-10-24_a%CC%80_13.31.49_hn4ibf.png)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    | Example of column schema and mapping in Google Sheets node.                                             |

---

This completes the detailed technical reference document for the “Scrape & Summarize Google Maps Businesses with APIFY + GPT-4O to Sheets” n8n workflow. It should enable users and AI agents to fully understand, reproduce, and modify the automation reliably.