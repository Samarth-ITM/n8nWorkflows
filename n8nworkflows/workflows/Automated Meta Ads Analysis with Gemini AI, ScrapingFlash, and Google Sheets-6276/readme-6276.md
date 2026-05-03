Automated Meta Ads Analysis with Gemini AI, ScrapingFlash, and Google Sheets

https://n8nworkflows.xyz/workflows/automated-meta-ads-analysis-with-gemini-ai--scrapingflash--and-google-sheets-6276


# Automated Meta Ads Analysis with Gemini AI, ScrapingFlash, and Google Sheets

### 1. Workflow Overview

This workflow automates the analysis of Meta (Facebook) ads by scraping ad data from Facebook Ad Library URLs, analyzing each ad using Google's Gemini AI model, and storing the structured AI-driven insights in a Google Sheet. It is designed to run on a schedule or be triggered manually, enabling marketing teams or analysts to efficiently monitor and evaluate ad creatives at scale.

The logical structure breaks down as follows:

- **1.1 Trigger & URL Acquisition:** Initiates the workflow either manually or on a schedule and fetches the list of Facebook Ad Library URLs from a Google Sheet.
- **1.2 Meta Ads Scraping:** Calls the ScrapingFlash API to scrape ads from each URL.
- **1.3 Ads Processing & Looping:** Splits the scraped ads into individual items and loops over them, limiting processing to 10 ads per run to manage costs.
- **1.4 AI Analysis with Gemini:** Sends each ad’s image and text to Google's Gemini AI for expert-level evaluation using a detailed prompt and structured output parsing.
- **1.5 Results Storage:** Appends the structured AI analysis output as new rows into a Google Sheet for record-keeping and further review.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger & URL Acquisition

**Overview:**  
This block starts the workflow either manually or on a schedule and retrieves a list of Facebook Ad Library URLs from a configured Google Sheet. These URLs serve as the input for scraping ads.

**Nodes Involved:**  
- When clicking ‘Execute workflow’ (Manual Trigger)  
- Schedule Trigger  
- Get URL to scrap (Google Sheets)  

**Node Details:**

- *When clicking ‘Execute workflow’*  
  - **Type:** Manual Trigger  
  - **Role:** Allows manual start of the workflow for testing or on-demand runs.  
  - **Config:** No parameters, triggers downstream nodes on user command.  
  - **Connections:** Outputs to "Get URL to scrap".  
  - **Edge Cases:** User must manually initiate; no automatic schedule. No expected failures.

- *Schedule Trigger*  
  - **Type:** Schedule Trigger  
  - **Role:** Automatically triggers the workflow on a repeating schedule (default is daily).  
  - **Config:** Interval set to daily (exact schedule can be adjusted).  
  - **Connections:** Outputs to "Get URL to scrap".  
  - **Edge Cases:** Workflow will not run if the n8n instance is down at scheduled time.

- *Get URL to scrap*  
  - **Type:** Google Sheets (Read operation)  
  - **Role:** Reads the Google Sheet containing the list of Facebook Ad Library URLs.  
  - **Config:** Configured with Google Sheets credentials, points to a specific spreadsheet and worksheet. Retrieves the column with URLs.  
  - **Connections:** Outputs to "Request to scrapingflash.com".  
  - **Edge Cases:** Possible errors if credentials expire, sheet or document ID is incorrect, or the sheet is empty.

---

#### 2.2 Meta Ads Scraping

**Overview:**  
This block sends the URLs obtained to the ScrapingFlash API to scrape ads from the Facebook Ad Library. The API returns a list of ads per URL.

**Nodes Involved:**  
- Request to scrapingflash.com (HTTP Request)  

**Node Details:**

- *Request to scrapingflash.com*  
  - **Type:** HTTP Request  
  - **Role:** Calls ScrapingFlash API with Facebook Ad Library URLs to retrieve ad data.  
  - **Config:** POST request to `https://api.scrapingflash.com/v1/facebook_ads`. Authentication via "Header Auth" credential storing the ScrapingFlash API key under header `x-api-key`.  
  - **Inputs:** Receives URLs from "Get URL to scrap".  
  - **Outputs:** Returns JSON containing ads under `body.ads`.  
  - **Edge Cases:** Authentication failure if key is invalid or missing; API rate limits; network timeouts; malformed URLs; empty or unexpected response format.

---

#### 2.3 Ads Processing & Looping

**Overview:**  
This block processes the scraped ads by splitting them into individual ad items and iterates over each ad separately. To control cost and runtime, the processing is limited to the first 10 ads per execution.

**Nodes Involved:**  
- Split all the ads (SplitOut)  
- Loop Over Items (SplitInBatches)  
- Limit to 10 ads (Limit)  

**Node Details:**

- *Split all the ads*  
  - **Type:** SplitOut  
  - **Role:** Splits the array of ads (from `body.ads`) into separate workflow items for individual processing.  
  - **Config:** Field to split out is `body.ads`.  
  - **Inputs:** Receives JSON array of ads from HTTP Request.  
  - **Outputs:** Emits one item per ad.  
  - **Edge Cases:** Empty or null ads array leads to no downstream processing.

- *Loop Over Items*  
  - **Type:** SplitInBatches  
  - **Role:** Processes items one by one to handle each ad sequentially.  
  - **Config:** Default batch size; processes ads individually.  
  - **Inputs:** Receives split ads from "Split all the ads".  
  - **Outputs:** Passes items to "Limit to 10 ads".  
  - **Edge Cases:** Large datasets will queue processing; batch size can be adjusted for performance.

- *Limit to 10 ads*  
  - **Type:** Limit  
  - **Role:** Restricts processing to the first 10 ads per workflow run to limit API costs and execution time.  
  - **Config:** Default limit of 10.  
  - **Inputs:** Receives ads from "Loop Over Items".  
  - **Outputs:** Sends limited set of ads to AI analysis.  
  - **Edge Cases:** Ads beyond 10 are ignored per run; can be adjusted or removed as needed.

---

#### 2.4 AI Analysis with Gemini

**Overview:**  
This core block analyzes each ad using Google's Gemini AI. It employs a LangChain Chain node with a detailed prompt instructing the AI to act as an expert Meta Ads analyst, evaluating the ad’s creative and text, scoring it, and suggesting improvements. The Structured Output Parser enforces a consistent JSON response format.

**Nodes Involved:**  
- Categorise every Meta Ads (LangChain Chain LLM)  
- Google Gemini Chat Model (LM Chat Google Gemini)  
- Structured Output Parser (LangChain Output Parser Structured)  

**Node Details:**

- *Categorise every Meta Ads*  
  - **Type:** LangChain Chain LLM  
  - **Role:** Sends ad data (image URL and text) to the Gemini Chat model with a complex prompt for ad evaluation. Parses and structures the response.  
  - **Config:**  
    - Text prompt instructs the AI to identify strengths, weaknesses, potential improvements, score the ad out of 5, and provide a TL;DR summary.  
    - Uses a custom-defined output schema for strict JSON parsing.  
    - Connects to Google Gemini Chat Model and Structured Output Parser nodes.  
  - **Inputs:** Receives individual ad items with image and text from "Limit to 10 ads".  
  - **Outputs:** Structured JSON analysis output.  
  - **Edge Cases:** AI response might not fully comply with schema, causing parse errors; model API quota or latency issues; malformed or missing ad data can degrade analysis quality.

- *Google Gemini Chat Model*  
  - **Type:** LangChain LM Chat Google Gemini  
  - **Role:** Hosts the Gemini AI model interaction, processing the prompt and returning the AI response.  
  - **Config:** Uses `models/gemini-pro-vision` model optimized for vision + text inputs.  
  - **Inputs:** Message and image URL from "Categorise every Meta Ads".  
  - **Outputs:** AI text response to be parsed.  
  - **Edge Cases:** Authentication failures, API rate limits, model errors, or latency.

- *Structured Output Parser*  
  - **Type:** LangChain Output Parser Structured  
  - **Role:** Parses Gemini AI output using a JSON schema to enforce consistent structured data.  
  - **Config:** JSON schema requires fields for ad image description, ad text, analysis (strengths, weaknesses, improvements), scoring (score and justification), and TL;DR.  
  - **Inputs:** Raw AI response from Google Gemini Chat Model.  
  - **Outputs:** Parsed JSON object for downstream use.  
  - **Edge Cases:** AI output not matching schema causes parse errors; incomplete or malformed AI responses.

---

#### 2.5 Results Storage

**Overview:**  
The final block appends the structured AI analysis data as new rows into a designated Google Sheet for record-keeping and further review.

**Nodes Involved:**  
- Add row in Sheet (Google Sheets)  

**Node Details:**

- *Add row in Sheet*  
  - **Type:** Google Sheets (Append operation)  
  - **Role:** Inserts each ad’s structured analysis as a new row in the configured Google Sheet.  
  - **Config:**  
    - Points to the Google Sheet and worksheet for storing results.  
    - Maps fields from the structured AI output to specific columns (e.g., strengths, weaknesses, score).  
    - Requires Google Sheets OAuth2 credentials.  
  - **Inputs:** Receives parsed structured analysis JSON from "Categorise every Meta Ads".  
  - **Outputs:** None (terminal node).  
  - **Edge Cases:** Credential expiration, quota limits, sheet access errors, or invalid mapping can cause failure.

---

### 3. Summary Table

| Node Name                | Node Type                           | Functional Role                      | Input Node(s)                 | Output Node(s)                 | Sticky Note                                                                                                                                                                                        |
|--------------------------|-----------------------------------|------------------------------------|------------------------------|-------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| README                   | Sticky Note                       | Documentation                      | -                            | -                             | ## AI Meta Ads Analyst\n\nDetailed explanation and usage instructions for the entire workflow.                                                                                                    |
| When clicking ‘Execute workflow’ | Manual Trigger                   | Manual workflow trigger             | -                            | Get URL to scrap               |                                                                                                                                                                                                   |
| Schedule Trigger         | Schedule Trigger                  | Scheduled automatic trigger         | -                            | Get URL to scrap               | ## 1. Trigger & Get URLs\n\nTriggers workflow and fetches Facebook Ad Library URLs from Google Sheets.                                                                                            |
| Sticky Note              | Sticky Note                       | Documentation                      | -                            | -                             | ## 1. Trigger & Get URLs\n\nExplains manual and scheduled triggers and fetching URLs from Google Sheets.                                                                                           |
| Get URL to scrap         | Google Sheets                    | Fetch URLs from Google Sheet        | When clicking ‘Execute workflow’, Schedule Trigger | Request to scrapingflash.com |                                                                                                                                                                                                   |
| Request to scrapingflash.com | HTTP Request                    | Scrape ads from URLs via API        | Get URL to scrap              | Split all the ads              | ## 2. Scrape Meta Ads\n\nCalls ScrapingFlash API to get ads. Requires Header Auth with API key.                                                                                                    |
| Sticky Note1             | Sticky Note                       | Documentation                      | -                            | -                             | ## 2. Scrape Meta Ads\n\nDescribes the ScrapingFlash API call and credential setup.                                                                                                               |
| Split all the ads        | SplitOut                         | Split ads array into individual ads | Request to scrapingflash.com  | Loop Over Items                |                                                                                                                                                                                                   |
| Loop Over Items          | SplitInBatches                   | Process ads one by one              | Split all the ads             | Limit to 10 ads               | ## 3. Process & Loop Through Ads\n\nExplains splitting ads and looping with limit to 10 for cost control.                                                                                          |
| Limit to 10 ads          | Limit                            | Limit number of ads processed       | Loop Over Items               | Categorise every Meta Ads      |                                                                                                                                                                                                   |
| Categorise every Meta Ads | LangChain Chain LLM              | AI analysis of each ad via Gemini   | Limit to 10 ads              | Add row in Sheet               | ## 4. Analyze Ad with Gemini AI\n\nSends ads to Gemini model with prompt, parses structured JSON output.                                                                                           |
| Sticky Note3             | Sticky Note                       | Documentation                      | -                            | -                             | ## 4. Analyze Ad with Gemini AI\n\nDetails about AI prompt and structured output parser.                                                                                                           |
| Google Gemini Chat Model | LangChain LM Chat Google Gemini | Gemini AI model interaction         | Categorise every Meta Ads    | Structured Output Parser       |                                                                                                                                                                                                   |
| Structured Output Parser | LangChain Output Parser Structured | Parses AI response into JSON       | Google Gemini Chat Model     | Categorise every Meta Ads      |                                                                                                                                                                                                   |
| Add row in Sheet         | Google Sheets                    | Append analysis results to sheet    | Categorise every Meta Ads    | -                             | ## 5. Save Analysis to Google Sheets\n\nAppends structured AI analysis as new rows in Google Sheet.                                                                                                |
| Sticky Note4             | Sticky Note                       | Documentation                      | -                            | -                             | ## 5. Save Analysis to Google Sheets\n\nInstructions for configuring Google Sheets node to save AI output.                                                                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes:**  
   - Add a **Manual Trigger** node named `When clicking ‘Execute workflow’` with no parameters.  
   - Add a **Schedule Trigger** node named `Schedule Trigger` configured to run daily (or as desired).  
   - Connect both triggers to the next node.

2. **Configure Google Sheets to Fetch URLs:**  
   - Add a **Google Sheets** node named `Get URL to scrap`.  
   - Operation: Read.  
   - Configure credentials for your Google account with Sheets access.  
   - Specify the target Sheet ID and worksheet containing Facebook Ad Library URLs.  
   - Select and map the column containing the URLs.  
   - Connect outputs of both trigger nodes to this node.

3. **Set up ScrapingFlash API Request:**  
   - Add an **HTTP Request** node named `Request to scrapingflash.com`.  
   - Method: POST.  
   - URL: `https://api.scrapingflash.com/v1/facebook_ads`.  
   - Authentication: Create a "Header Auth" credential with header name `x-api-key` and value set to your ScrapingFlash API key.  
   - Pass the URLs from "Get URL to scrap" to this node’s body or parameters as API expects.  
   - Connect output of `Get URL to scrap` to this node.

4. **Split Ads Array:**  
   - Add a **SplitOut** node named `Split all the ads`.  
   - Configure to split on the field `body.ads` (the array of ads returned by ScrapingFlash).  
   - Connect output of HTTP Request node to this node.

5. **Loop Through Ads One by One:**  
   - Add a **SplitInBatches** node named `Loop Over Items`.  
   - Use default batch size to process ads individually.  
   - Connect output of `Split all the ads` to this node.

6. **Limit Number of Ads Processed:**  
   - Add a **Limit** node named `Limit to 10 ads` with the default limit set to 10.  
   - Connect output of `Loop Over Items` to this node.

7. **Set Up LangChain AI Analysis:**  
   - Add a **LangChain Chain LLM** node named `Categorise every Meta Ads`.  
   - Configure the prompt to instruct the AI to analyze Meta ads as per the detailed prompt in the workflow:  
     - Include analysis of strengths, weaknesses, suggestions, scoring (out of 5), and TL;DR summary.  
   - Set the input variables to receive ad image URL and ad text.  
   - Enable output parser with a **Structured Output Parser** node configured with the provided JSON schema requiring analysis fields.  
   - Connect output of `Limit to 10 ads` to this node.

8. **Configure Gemini AI Model:**  
   - Add a **LangChain LM Chat Google Gemini** node named `Google Gemini Chat Model`.  
   - Set model name to `models/gemini-pro-vision`.  
   - Connect this as the AI language model for the `Categorise every Meta Ads` node.

9. **Configure Structured Output Parser:**  
   - Add a **LangChain Output Parser Structured** node named `Structured Output Parser`.  
   - Paste the JSON schema defining required output structure (ad_image, ad_text, analysis, scoring, tldr).  
   - Connect this parser to the `Categorise every Meta Ads` node.

10. **Append Results to Google Sheets:**  
    - Add a **Google Sheets** node named `Add row in Sheet`.  
    - Operation: Append.  
    - Configure credentials (same or different Google account).  
    - Set the target Sheet and worksheet for results storage.  
    - Map the structured fields from the AI output (strengths, weaknesses, score, justification, TL;DR, etc.) to columns.  
    - Connect output of `Categorise every Meta Ads` to this node.

11. **Final Connections:**  
    - Connect all nodes as per the flow:  
      - Triggers → Get URL to scrap → Request to scrapingflash.com → Split all the ads → Loop Over Items → Limit to 10 ads → Categorise every Meta Ads → Add row in Sheet.

12. **Credential Setup:**  
    - Create and configure:  
      - Google Sheets OAuth2 credentials for reading and writing sheets.  
      - ScrapingFlash API key credential as Header Auth (`x-api-key`).  
      - Google Gemini API key credential for LangChain Gemini nodes.  

13. **Testing and Activation:**  
    - Test manually by clicking “Execute Workflow”.  
    - Once confirmed, activate the workflow to run automatically on schedule.

---

### 5. General Notes & Resources

| Note Content                                                                                                                       | Context or Link                                                            |
|-----------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------|
| Join the Discord community for help and discussion on this workflow.                                                              | https://discord.com/invite/XPKeKXeB7d                                    |
| Ask questions and find support on the official n8n Forum.                                                                          | https://community.n8n.io/                                                  |
| Before running, ensure all credentials (ScrapingFlash, Google Gemini, Google Sheets) are correctly configured and authorized.     | Workflow README sticky note                                               |
| The workflow limits AI processing to 10 ads per run by default to manage costs; adjust the Limit node as needed for scale.        | Sticky Note on “Limit to 10 ads” node                                    |
| The AI analysis prompt and output schema are tailored for consistent and actionable ad evaluation results.                         | Sticky Note on “Categorise every Meta Ads” and “Structured Output Parser”|

---

This documentation fully describes the "Automated Meta Ads Analysis with Gemini AI, ScrapingFlash, and Google Sheets" workflow, enabling replication, modification, and troubleshooting by both humans and automation agents.