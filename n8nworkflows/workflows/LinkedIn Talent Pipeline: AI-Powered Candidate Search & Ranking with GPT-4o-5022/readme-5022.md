LinkedIn Talent Pipeline: AI-Powered Candidate Search & Ranking with GPT-4o

https://n8nworkflows.xyz/workflows/linkedin-talent-pipeline--ai-powered-candidate-search---ranking-with-gpt-4o-5022


# LinkedIn Talent Pipeline: AI-Powered Candidate Search & Ranking with GPT-4o

### 1. Workflow Overview

This workflow titled **"LinkedIn Talent Pipeline: AI-Powered Candidate Search & Ranking with GPT-4o"** automates the process of sourcing, extracting, and ranking candidate profiles for recruitment purposes using LinkedIn data. Its primary goal is to transform a job description input into a structured list of ranked LinkedIn candidate profiles saved to Google Sheets, leveraging AI (GPT-4o) for intelligent parsing and scoring.

The workflow is composed of the following logical blocks:

- **1.1 Job Description Reception & Parsing:** Accepts job descriptions via webhook, uses AI to extract structured job details.
- **1.2 Search Query Generation & Candidate URL Retrieval:** Constructs search page numbers, performs Google Custom Search to find LinkedIn profiles, filters relevant profile URLs.
- **1.3 Candidate Data Scraping & Monitoring:** Prepares input for Apify scraper, triggers scraping, waits and checks for completion, retrieves scraped LinkedIn data.
- **1.4 AI-Based Candidate Scoring:** Scores candidates using AI models based on scraped data.
- **1.5 Result Processing & Storage:** Parses AI scoring results and stores ranked candidates in Google Sheets.
- **1.6 Workflow Support & Memory:** Uses in-memory buffers and structured output parsers to maintain context and improve parsing accuracy.

---

### 2. Block-by-Block Analysis

#### 2.1 Job Description Reception & Parsing

- **Overview:** This block begins the workflow by receiving a job description through a webhook trigger and uses AI to parse and extract detailed structured job information for downstream processing.

- **Nodes Involved:**
  - Job Description Input
  - OpenAI Chat Model
  - Window Buffer Memory
  - Extract Job Details
  - Structured Output Parser
  - Set Search Page Numbers
  - Split Search Pages

- **Node Details:**

1. **Job Description Input**
   - *Type:* LangChain Chat Trigger (webhook)
   - *Role:* Entry point for receiving job descriptions via webhook.
   - *Configuration:* Triggered by an external HTTP request; no additional parameters.
   - *Connections:* Outputs to Extract Job Details.
   - *Edge Cases:* Missing or malformed webhook payload; timeout if no input.
   - *Version:* 1.1

2. **OpenAI Chat Model**
   - *Type:* LangChain OpenAI Chat Model
   - *Role:* Provides the language model for parsing job descriptions.
   - *Configuration:* Uses OpenAI GPT model (likely GPT-4o), configured with appropriate credentials.
   - *Connections:* Feeds into Extract Job Details via ai_languageModel input.
   - *Edge Cases:* API rate limits, authentication failures, or model timeout.
   - *Version:* 1

3. **Window Buffer Memory**
   - *Type:* LangChain Buffer Memory
   - *Role:* Maintains conversational context for the AI agent to improve parsing.
   - *Configuration:* Window buffer keeps recent inputs; no parameters exposed.
   - *Connections:* Connected to Extract Job Details via ai_memory.
   - *Version:* 1.3

4. **Extract Job Details**
   - *Type:* LangChain Agent
   - *Role:* Uses the AI model and memory to extract structured job details from the input text.
   - *Configuration:* Agent node configured to run the parsing logic.
   - *Connections:* Outputs to Set Search Page Numbers.
   - *Version:* 1.7
   - *Edge Cases:* AI misinterpretation or parsing errors.

5. **Structured Output Parser**
   - *Type:* LangChain Output Parser Structured
   - *Role:* Parses AI output into a structured format.
   - *Connections:* Feeds into Extract Job Details (ai_outputParser).
   - *Version:* 1.2

6. **Set Search Page Numbers**
   - *Type:* Set
   - *Role:* Defines the range or list of page numbers to use for Google Custom Search pagination.
   - *Configuration:* Likely sets an array or range of integers for pages to search.
   - *Connections:* Outputs to Split Search Pages.
   - *Version:* 3.4

7. **Split Search Pages**
   - *Type:* Split Out
   - *Role:* Splits the list of page numbers into individual items for iterative API requests.
   - *Connections:* Outputs to Google Custom Search API Request.
   - *Version:* 1

---

#### 2.2 Search Query Generation & Candidate URL Retrieval

- **Overview:** This block performs Google Custom Searches using the job details to find LinkedIn profiles, then filters and prepares the candidate URLs for scraping.

- **Nodes Involved:**
  - Google Custom Search API Request
  - Filter LinkedIn Profiles
  - Prepare Apify Input
  - Extract Profile URLs
  - Limit

- **Node Details:**

1. **Google Custom Search API Request**
   - *Type:* HTTP Request
   - *Role:* Queries Google Custom Search API with search parameters derived from job details.
   - *Configuration:* Configured with API endpoint, query parameters including job keywords and page numbers.
   - *Connections:* Outputs to Filter LinkedIn Profiles.
   - *Edge Cases:* API quota limits, request failures, invalid query parameters.
   - *Version:* 4.2

2. **Filter LinkedIn Profiles**
   - *Type:* Function (JavaScript)
   - *Role:* Filters the search results to retain only LinkedIn profile URLs.
   - *Configuration:* Script inspects the response items and filters URLs based on LinkedIn domain patterns.
   - *Connections:* Outputs to Prepare Apify Input.
   - *Version:* 1

3. **Prepare Apify Input**
   - *Type:* Function
   - *Role:* Prepares input data format required by Apify scraper from filtered LinkedIn profile URLs.
   - *Configuration:* Transforms URLs into the JSON or payload format expected by Apify API.
   - *Connections:* Outputs to Extract Profile URLs.
   - *Version:* 1

4. **Extract Profile URLs**
   - *Type:* Code
   - *Role:* Further processes or extracts LinkedIn URLs from the prepared input.
   - *Connections:* Outputs to Limit node.
   - *Version:* 2

5. **Limit**
   - *Type:* Limit
   - *Role:* Restricts the number of LinkedIn profiles processed in the batch to control workload.
   - *Configuration:* Likely sets a maximum number of items per batch.
   - *Connections:* Outputs to Loop Over Items.
   - *Version:* 1

---

#### 2.3 Candidate Data Scraping & Monitoring

- **Overview:** This block coordinates running the Apify scraping jobs on LinkedIn profiles, waits for scraping completion, and retrieves scraped candidate data.

- **Nodes Involved:**
  - Loop Over Items
  - Run Apify Scraper
  - Wait for Scraping
  - Check Scraping Status
  - Verify Completion
  - Get LinkedIn Data

- **Node Details:**

1. **Loop Over Items**
   - *Type:* Split In Batches
   - *Role:* Processes LinkedIn profile URLs in manageable batch sizes.
   - *Connections:* 
     - Main output 0: empty (end of batch)
     - Main output 1: Run Apify Scraper (for each batch)
   - *Version:* 3

2. **Run Apify Scraper**
   - *Type:* HTTP Request
   - *Role:* Initiates scraping of LinkedIn profiles via Apify API.
   - *Configuration:* Sends scraping job requests with profile URLs.
   - *Connections:* Outputs to Wait for Scraping.
   - *Version:* 4.2

3. **Wait for Scraping**
   - *Type:* Wait
   - *Role:* Pauses workflow execution to allow scraping jobs to complete.
   - *Configuration:* Waits for a webhook or a fixed timeout.
   - *Connections:* Outputs to Check Scraping Status.
   - *Version:* 1.1

4. **Check Scraping Status**
   - *Type:* HTTP Request
   - *Role:* Polls Apify API to check if scraping jobs have completed.
   - *Connections:* Outputs to Verify Completion.
   - *Version:* 4.2

5. **Verify Completion**
   - *Type:* If
   - *Role:* Determines if scraping is complete or if the workflow should wait longer.
   - *Connections:*
     - True output: Get LinkedIn Data
     - False output: Wait for Scraping (loop)
   - *Version:* 2.2

6. **Get LinkedIn Data**
   - *Type:* HTTP Request
   - *Role:* Retrieves scraped LinkedIn profile data from Apify.
   - *Connections:* Outputs to LLM Scoring.
   - *Version:* 4.2

---

#### 2.4 AI-Based Candidate Scoring

- **Overview:** This block applies AI scoring to the scraped LinkedIn profile data to rank candidates based on relevance to the job description.

- **Nodes Involved:**
  - LLM Scoring
  - Parse LLM Result

- **Node Details:**

1. **LLM Scoring**
   - *Type:* LangChain OpenAI
   - *Role:* Uses an OpenAI language model to score candidates.
   - *Configuration:* Likely uses GPT-4o with scoring prompts tailored to candidate evaluation.
   - *Connections:* Outputs to Parse LLM Result.
   - *Edge Cases:* Model failures, API limits.
   - *Version:* 1.7

2. **Parse LLM Result**
   - *Type:* Code
   - *Role:* Parses AI scoring results into usable data structures.
   - *Connections:* Outputs to Save to Google Sheets.
   - *Version:* 2

---

#### 2.5 Result Processing & Storage

- **Overview:** This block stores the scored candidate rankings into a Google Sheet for easy access and further analysis.

- **Nodes Involved:**
  - Save to Google Sheets

- **Node Details:**

1. **Save to Google Sheets**
   - *Type:* Google Sheets
   - *Role:* Writes candidate ranking data into a configured Google Sheets document.
   - *Configuration:* Requires Google Sheets credentials and target spreadsheet details.
   - *Connections:* Outputs to Loop Over Items (likely to iterate batches).
   - *Edge Cases:* Authentication errors, rate limits, spreadsheet access errors.
   - *Version:* 4.5

---

#### 2.6 Workflow Support & Memory

- **Overview:** This block contains sticky notes and other nodes for documentation and workflow memory management.

- **Nodes Involved:**
  - Workflow Documentation (Sticky Note)
  - Sticky Note
  - Sticky Note1

- **Node Details:**

1. **Workflow Documentation / Sticky Note / Sticky Note1**
   - *Type:* Sticky Note
   - *Role:* Provide inline documentation or reminders; no functional impact.
   - *Configuration:* Content currently empty.
   - *Version:* 1

---

### 3. Summary Table

| Node Name                   | Node Type                                   | Functional Role                          | Input Node(s)                | Output Node(s)                   | Sticky Note              |
|-----------------------------|---------------------------------------------|----------------------------------------|-----------------------------|---------------------------------|--------------------------|
| Job Description Input        | LangChain Chat Trigger                      | Entry webhook for job descriptions     |                             | Extract Job Details              |                          |
| OpenAI Chat Model            | LangChain OpenAI Chat Model                 | Provides AI model for parsing          |                             | Extract Job Details (ai_languageModel) |                          |
| Window Buffer Memory         | LangChain Memory Buffer Window              | Maintains AI conversational context    |                             | Extract Job Details (ai_memory) |                          |
| Extract Job Details          | LangChain Agent                             | Extracts structured job details        | Job Description Input, OpenAI Chat Model, Window Buffer Memory, Structured Output Parser | Set Search Page Numbers           |                          |
| Structured Output Parser     | LangChain Output Parser Structured          | Parses AI output into structured data  |                             | Extract Job Details (ai_outputParser) |                          |
| Set Search Page Numbers      | Set                                         | Defines pagination pages for search    | Extract Job Details          | Split Search Pages              |                          |
| Split Search Pages           | Split Out                                   | Splits page numbers into separate items| Set Search Page Numbers      | Google Custom Search API Request |                          |
| Google Custom Search API Request | HTTP Request                             | Performs Google Custom Search          | Split Search Pages           | Filter LinkedIn Profiles         |                          |
| Filter LinkedIn Profiles     | Function                                    | Filters for LinkedIn URLs               | Google Custom Search API Request | Prepare Apify Input            |                          |
| Prepare Apify Input          | Function                                    | Prepares input for Apify scraper        | Filter LinkedIn Profiles     | Extract Profile URLs            |                          |
| Extract Profile URLs         | Code                                        | Extracts LinkedIn URLs                  | Prepare Apify Input          | Limit                          |                          |
| Limit                       | Limit                                       | Limits number of profiles processed    | Extract Profile URLs         | Loop Over Items                |                          |
| Loop Over Items             | Split In Batches                             | Processes profiles in batches           | Limit                       | Run Apify Scraper               |                          |
| Run Apify Scraper            | HTTP Request                                | Initiates Apify scraping jobs           | Loop Over Items             | Wait for Scraping              |                          |
| Wait for Scraping            | Wait                                        | Waits for scraping to complete          | Run Apify Scraper, Verify Completion (false branch) | Check Scraping Status          |                          |
| Check Scraping Status        | HTTP Request                                | Polls Apify for scraping job status     | Wait for Scraping            | Verify Completion              |                          |
| Verify Completion            | If                                          | Checks if scraping completed            | Check Scraping Status        | Get LinkedIn Data (true), Wait for Scraping (false) |                          |
| Get LinkedIn Data            | HTTP Request                                | Retrieves scraped LinkedIn profile data | Verify Completion (true)     | LLM Scoring                   |                          |
| LLM Scoring                 | LangChain OpenAI                             | Scores candidates using AI               | Get LinkedIn Data            | Parse LLM Result              |                          |
| Parse LLM Result             | Code                                        | Parses AI scoring output                 | LLM Scoring                 | Save to Google Sheets         |                          |
| Save to Google Sheets        | Google Sheets                               | Stores candidate rankings                | Parse LLM Result             | Loop Over Items               |                          |
| Workflow Documentation       | Sticky Note                                 | Notes/documentation                      |                             |                                 |                          |
| Sticky Note                 | Sticky Note                                  | Notes/documentation                      |                             |                                 |                          |
| Sticky Note1                | Sticky Note                                  | Notes/documentation                      |                             |                                 |                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "Job Description Input" node**
   - Type: LangChain Chat Trigger (Webhook)
   - Configure webhook with unique ID (e.g., "job-desc-webhook")
   - No parameters needed; this receives job description text.

2. **Create "OpenAI Chat Model" node**
   - Type: LangChain OpenAI Chat Model
   - Configure with OpenAI credentials (API key)
   - Choose GPT-4o or equivalent model
   - No custom parameters required.

3. **Create "Window Buffer Memory" node**
   - Type: LangChain Memory Buffer Window
   - Use default buffer window settings
   - No configuration needed.

4. **Create "Structured Output Parser" node**
   - Type: LangChain Output Parser Structured
   - Configure to parse AI agent outputs into structured JSON.

5. **Create "Extract Job Details" node**
   - Type: LangChain Agent
   - Connect inputs:
     - ai_languageModel: OpenAI Chat Model
     - ai_memory: Window Buffer Memory
     - ai_outputParser: Structured Output Parser
     - main input: Job Description Input
   - No additional parameters; logic runs based on connected AI nodes.

6. **Create "Set Search Page Numbers" node**
   - Type: Set
   - Add a field setting page numbers (e.g., an array [1, 2, 3, 4, 5]) to paginate Google search.

7. **Create "Split Search Pages" node**
   - Type: Split Out
   - Connect main input from Set Search Page Numbers
   - No further config; splits array for iterative processing.

8. **Create "Google Custom Search API Request" node**
   - Type: HTTP Request
   - Configure with Google Custom Search API endpoint
   - Set query parameters including job keywords, current page number (from Split Search Pages)
   - Connect input from Split Search Pages.

9. **Create "Filter LinkedIn Profiles" node**
   - Type: Function
   - Write JavaScript to filter search results for URLs containing "linkedin.com/in"
   - Connect input from Google Custom Search API Request.

10. **Create "Prepare Apify Input" node**
    - Type: Function
    - Transform filtered URLs into JSON payloads for Apify scraping input.
    - Connect input from Filter LinkedIn Profiles.

11. **Create "Extract Profile URLs" node**
    - Type: Code
    - Further extract or clean LinkedIn URLs.
    - Connect input from Prepare Apify Input.

12. **Create "Limit" node**
    - Type: Limit
    - Set max number of profiles per batch (e.g., 20)
    - Connect input from Extract Profile URLs.

13. **Create "Loop Over Items" node**
    - Type: Split In Batches
    - Set batch size (e.g., 5)
    - Connect input from Limit.

14. **Create "Run Apify Scraper" node**
    - Type: HTTP Request
    - Configure Apify API endpoint to start scraping job
    - Use credentials or API token for Apify
    - Connect input from Loop Over Items (batch output).

15. **Create "Wait for Scraping" node**
    - Type: Wait
    - Configure with webhook ID or fixed delay (e.g., 1 minute)
    - Connect input from Run Apify Scraper.

16. **Create "Check Scraping Status" node**
    - Type: HTTP Request
    - Poll Apify API for job status
    - Connect input from Wait for Scraping.

17. **Create "Verify Completion" node**
    - Type: If
    - Condition: Check if scraping status is "COMPLETED"
    - Connect input from Check Scraping Status.
    - True output to Get LinkedIn Data.
    - False output loops back to Wait for Scraping.

18. **Create "Get LinkedIn Data" node**
    - Type: HTTP Request
    - Retrieve scraped candidate data from Apify
    - Connect input from Verify Completion (true output).

19. **Create "LLM Scoring" node**
    - Type: LangChain OpenAI
    - Configure with OpenAI credentials and scoring prompt template
    - Connect input from Get LinkedIn Data.

20. **Create "Parse LLM Result" node**
    - Type: Code
    - Parse AI scoring results for structured ranking.
    - Connect input from LLM Scoring.

21. **Create "Save to Google Sheets" node**
    - Type: Google Sheets
    - Configure with Google OAuth2 credentials
    - Set target spreadsheet and worksheet
    - Connect input from Parse LLM Result.
    - Output connects back to Loop Over Items to process next batch.

22. **Add Sticky Notes**
    - Add Sticky Note nodes for workflow documentation or reminders as needed.

---

### 5. General Notes & Resources

| Note Content                                                                                   | Context or Link                                               |
|------------------------------------------------------------------------------------------------|---------------------------------------------------------------|
| The workflow leverages GPT-4o for advanced NLP parsing and scoring of candidate profiles.      | AI model configuration nodes (OpenAI Chat Model, LLM Scoring) |
| Apify scraping integration requires valid Apify API credentials and proper scraper setup.      | Apify API Request nodes (Run Apify Scraper, Check Scraping Status, Get LinkedIn Data) |
| Google Custom Search API usage is subject to quota limits; ensure API key and CX ID are set.   | Google Custom Search API Request node                          |
| Google Sheets node requires OAuth2 credentials with write access to the target spreadsheet.    | Save to Google Sheets node                                     |
| Pagination control via "Set Search Page Numbers" allows flexible search depth tuning.           | Set Search Page Numbers node                                   |
| The "Wait for Scraping" node uses webhook ID to resume workflow upon scraper completion webhook.| Wait node configuration                                       |

---

**Disclaimer:**  
The provided content is extracted exclusively from an n8n automated workflow. This process complies fully with applicable content policies and contains no illegal or protected information. All data handled is legal and publicly accessible.