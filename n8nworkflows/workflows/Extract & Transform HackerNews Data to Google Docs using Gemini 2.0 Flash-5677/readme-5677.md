Extract & Transform HackerNews Data to Google Docs using Gemini 2.0 Flash

https://n8nworkflows.xyz/workflows/extract---transform-hackernews-data-to-google-docs-using-gemini-2-0-flash-5677


# Extract & Transform HackerNews Data to Google Docs using Gemini 2.0 Flash

### 1. Workflow Overview

This workflow automates the extraction of structured data from Hacker News, processes it through Google Gemini 2.0 Flash AI to generate a human-readable summary, and exports the results into Google Docs. It is designed to fetch a configurable number of Hacker News items, enrich them with AI-driven summaries, and save them into individual Google Document files for easy review and further sharing.

The workflow is logically divided into these functional blocks:

- **1.1 Trigger and Input Setup:** Manual execution trigger and setting the number of Hacker News records to fetch.
- **1.2 Hacker News Data Retrieval:** Querying Hacker News API to fetch top stories based on the configured count.
- **1.3 Data Loop and Preparation:** Iterating through each fetched story to prepare and enrich data fields such as author and URL.
- **1.4 HTTP Content Fetch:** Retrieving the web content of each Hacker News item via HTTP request to provide context for AI processing.
- **1.5 AI Processing with Google Gemini:** Using Google Gemini 2.0 Flash model to extract a clean, human-readable summary from the raw data.
- **1.6 Google Docs Export:** Creating and updating Google Docs documents with the AI-processed summaries for each Hacker News item.

This modular design ensures clear data flow, error handling opportunities, and easy extensibility for other data sources or output formats.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger and Input Setup

**Overview:**  
This block initiates the workflow manually and sets the key input parameter, the number of Hacker News records to fetch.

**Nodes Involved:**  
- When clicking ‘Execute workflow’  
- Set the Input Fields  
- Sticky Note (Step 1)

**Node Details:**

- **When clicking ‘Execute workflow’**  
  - Type: Manual Trigger  
  - Role: Starts workflow execution on user command  
  - Config: Default manual trigger settings  
  - Inputs/Outputs: None input; outputs to "Set the Input Fields"  
  - Failures: None typical; workflow does not start if node is disabled

- **Set the Input Fields**  
  - Type: Set  
  - Role: Defines the input parameter named "Count" with a default value of "5" (string)  
  - Config: Assigns Count = "5" (string)  
  - Expressions: None dynamic; static assignment  
  - Inputs: From manual trigger  
  - Outputs: To "Hacker News" node  
  - Edge cases: If "Count" is not a valid number, downstream nodes may fail or return no data

- **Sticky Note (Step 1)**  
  - Type: Sticky Note  
  - Content: Explains that this step sets the count of records to fetch  
  - Position: Visually associated with input setup nodes

---

#### 2.2 Hacker News Data Retrieval

**Overview:**  
Fetches a specified number of Hacker News records using the Hacker News API node.

**Nodes Involved:**  
- Hacker News  
- Loop Over Items  
- Sticky Note4 ("Hacker News Data Extraction")

**Node Details:**

- **Hacker News**  
  - Type: Hacker News API node  
  - Role: Retrieves top stories limited by the "Count" parameter  
  - Config:  
    - Resource: all (fetches all types of Hacker News stories)  
    - Limit: Uses expression `={{ $json.Count }}` to dynamically fetch count from input  
  - Inputs: From "Set the Input Fields"  
  - Outputs: Array of Hacker News items  
  - Failures: API rate limits, connectivity issues, or invalid limits may cause failure  
  - Notes: Data returned includes metadata like author, URL, etc.

- **Loop Over Items**  
  - Type: SplitInBatches  
  - Role: Processes each Hacker News item individually to enable sequential data enrichment  
  - Config: Default batching (one at a time)  
  - Inputs: From "Hacker News"  
  - Outputs: To "Set the url, author"  
  - Failures: Batch size misconfiguration can cause performance degradation

- **Sticky Note4**  
  - Explains this block handles Hacker News data extraction  

---

#### 2.3 Data Loop and Preparation

**Overview:**  
For each Hacker News item, extracts author and URL fields, then fetches the content from the URL for AI processing.

**Nodes Involved:**  
- Set the url, author  
- Create an HTTP Request  
- Sticky Note1 (Google Gemini Credentials)  
- Sticky Note5 (LLM Usage info)

**Node Details:**

- **Set the url, author**  
  - Type: Set  
  - Role: Extracts and renames key fields "author" and "url" from the Hacker News item JSON for clarity  
  - Config:  
    - author = `{{$json.author}}`  
    - url = `{{$json._highlightResult.url.value}}` (uses highlight result for URL)  
  - Inputs: From "Loop Over Items" (output 1)  
  - Outputs: To "Create an HTTP Request"  
  - Edge cases: If fields are missing or null, downstream HTTP request may fail

- **Create an HTTP Request**  
  - Type: HTTP Request  
  - Role: Fetches content from the URL extracted from Hacker News to provide context for AI processing  
  - Config:  
    - URL: `={{ $json.url }}` (dynamic URL from previous node)  
    - No additional options set  
  - Inputs: From "Set the url, author"  
  - Outputs: To "Extract Human Readable Data"  
  - Failures: HTTP errors such as 404, timeout, invalid URL, or connectivity issues may occur; retry enabled  
  - Notes: Outputs raw web content for AI to analyze

- **Sticky Note1**  
  - Indicates Google Gemini credentials are needed for the next AI processing step

- **Sticky Note5**  
  - Notes the use of Google Gemini 2.0 Flash Exp model for LLM tasks  

---

#### 2.4 AI Processing with Google Gemini

**Overview:**  
Processes the raw web content using Google Gemini AI to extract a clean, human-readable summary.

**Nodes Involved:**  
- Google Gemini Chat Model  
- Extract Human Readable Data  
- Sticky Note (Step 2)

**Node Details:**

- **Google Gemini Chat Model**  
  - Type: LangChain Google Gemini Language Model node  
  - Role: AI model integration providing language understanding  
  - Config:  
    - Model name: "models/gemini-2.0-flash-exp"  
    - No additional options set  
  - Credentials: Google Palm API credentials required  
  - Inputs: AI language model input from "Extract Human Readable Data" node (ai_languageModel connection)  
  - Outputs: AI-generated textual data  
  - Failures: Invalid credentials, API limits, or service downtime may cause failure

- **Extract Human Readable Data**  
  - Type: LangChain Chain LLM node  
  - Role: Defines prompt to extract human-readable content from raw data  
  - Config:  
    - Prompt text includes: "Extract a human readable content. Here's the context: {{ $json.data }}"  
    - Retry on fail enabled  
  - Inputs: HTTP request output (raw content), LLM model connection from Gemini Chat Model  
  - Outputs: To "Create a Google Doc"  
  - Edge cases: Prompt misconfiguration or input data invalidity can lead to poor output or errors

- **Sticky Note (Step 2)**  
  - Reminds to configure Google Gemini credentials properly  

---

#### 2.5 Google Docs Export

**Overview:**  
Creates a new Google Document for each Hacker News item and inserts the AI-processed human-readable summary into it.

**Nodes Involved:**  
- Create a Google Doc  
- Update Google Docs  
- Sticky Note3 (Google Docs Credentials)  

**Node Details:**

- **Create a Google Doc**  
  - Type: Google Docs node (create)  
  - Role: Creates a new Google Document with the author's name as the title prefix  
  - Config:  
    - Title: `={{ $('Set the url, author').item.json.author }}-HackerNews` (dynamic title)  
    - Folder: Default folder  
  - Credentials: Google Docs OAuth2 credentials required  
  - Inputs: From "Extract Human Readable Data"  
  - Outputs: To "Update Google Docs"  
  - Retry on fail enabled  
  - Edge cases: Permissions issues, quota limits, or invalid folder ID can cause failure

- **Update Google Docs**  
  - Type: Google Docs node (update)  
  - Role: Inserts the AI-generated text summary into the newly created Google Document  
  - Config:  
    - Operation: update  
    - Document URL: `={{ $json.id }}` (document ID from creation)  
    - Actions: Insert text from `={{ $('Extract Human Readable Data').item.json.text }}`  
  - Credentials: Same Google Docs OAuth2 credentials  
  - Inputs: From "Create a Google Doc"  
  - Outputs: Loops back to "Loop Over Items" (main output 1) to process next batch item  
  - Edge cases: Document access errors, invalid document ID, or insertion failure possible

- **Sticky Note3**  
  - Instructs to set Google Document credentials for export  

---

### 3. Summary Table

| Node Name                 | Node Type                          | Functional Role                             | Input Node(s)            | Output Node(s)               | Sticky Note                                      |
|---------------------------|----------------------------------|---------------------------------------------|--------------------------|-----------------------------|-------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger                   | Starts workflow manually                    | -                        | Set the Input Fields         |                                                 |
| Set the Input Fields       | Set                              | Sets number of records to fetch ("Count")  | When clicking ‘Execute workflow’ | Hacker News                 | Step 1: Set the input field with the "Count".   |
| Hacker News               | Hacker News API                   | Fetches Hacker News data                     | Set the Input Fields      | Loop Over Items              | Hacker News Data Extraction                      |
| Loop Over Items            | SplitInBatches                   | Iterates through Hacker News items          | Hacker News              | Set the url, author          | Hacker News Data Extraction                      |
| Set the url, author        | Set                              | Extracts author and URL from item            | Loop Over Items          | Create an HTTP Request       |                                                 |
| Create an HTTP Request     | HTTP Request                     | Fetches content from Hacker News URL         | Set the url, author      | Extract Human Readable Data  |                                                 |
| Google Gemini Chat Model   | LangChain Google Gemini LLM      | Provides AI model for text extraction        | Extract Human Readable Data (ai_languageModel) | Extract Human Readable Data (ai_languageModel) | Step 2: Set Google Gemini Credentials            |
| Extract Human Readable Data | LangChain Chain LLM              | Extracts human-readable summary from content| Create an HTTP Request   | Create a Google Doc          | Step 2: Set Google Gemini Credentials            |
| Create a Google Doc        | Google Docs (create)             | Creates new Google Document                   | Extract Human Readable Data | Update Google Docs          | Step 3: Set Google Docs Credentials               |
| Update Google Docs         | Google Docs (update)             | Inserts AI text into Google Document          | Create a Google Doc       | Loop Over Items             | Step 3: Set Google Docs Credentials               |
| Sticky Note (Step 1)       | Sticky Note                     | Notes input setup step                        | -                        | -                           | Step 1: Set the input field with the "Count".   |
| Sticky Note1 (Step 2)      | Sticky Note                     | Notes Google Gemini credential setup          | -                        | -                           | Step 2: Set Google Gemini Credentials             |
| Sticky Note2               | Sticky Note                     | Workflow purpose summary                      | -                        | -                           | Extract Structured Data from Hacker News workflow|
| Sticky Note3 (Step 3)      | Sticky Note                     | Notes Google Docs credential setup            | -                        | -                           | Step 3: Set Google Docs Credentials               |
| Sticky Note4               | Sticky Note                     | Notes Hacker News data extraction             | -                        | -                           | Hacker News Data Extraction                      |
| Sticky Note5               | Sticky Note                     | Notes LLM usage details                        | -                        | -                           | LLM Usages: Google Gemini 2.0 Flash Exp Model    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Purpose: Start workflow manually  
   - No special config needed

2. **Create Set Node ("Set the Input Fields")**  
   - Connect from Manual Trigger  
   - Add field: Name = "Count", Type = String, Value = "5" (default number of records to fetch)  

3. **Create Hacker News Node**  
   - Connect from "Set the Input Fields"  
   - Resource: "all" (fetch all stories)  
   - Limit: Use expression referencing `{{$json.Count}}` from previous node  
   - No additional fields needed

4. **Create SplitInBatches Node ("Loop Over Items")**  
   - Connect from Hacker News node  
   - Use default batch size (1) to process one item at a time

5. **Create Set Node ("Set the url, author")**  
   - Connect from second output (batch output) of SplitInBatches  
   - Set fields:  
     - author = `{{$json.author}}`  
     - url = `{{$json._highlightResult.url.value}}` (or fallback to `$json.url` if highlight not available)

6. **Create HTTP Request Node**  
   - Connect from "Set the url, author"  
   - Method: GET (default)  
   - URL: `{{$json.url}}` (dynamic)  
   - Enable "Retry on Fail" for robustness

7. **Create Google Gemini Chat Model Node**  
   - Set model name: "models/gemini-2.0-flash-exp"  
   - Add Google Palm API credentials (configure OAuth or API key)  
   - No special parameters needed

8. **Create LangChain Chain LLM Node ("Extract Human Readable Data")**  
   - Connect HTTP Request main output to this node  
   - Connect AI language model input to Google Gemini Chat Model node  
   - Set prompt with text:  
     ```
     Extract a human readable content. 

     Here's the context :

     {{ $json.data }}
     ```  
   - Enable "Retry on Fail" for reliability

9. **Create Google Docs Node ("Create a Google Doc")**  
   - Connect from "Extract Human Readable Data"  
   - Operation: Create  
   - Title: Use expression `={{ $('Set the url, author').item.json.author }}-HackerNews`  
   - Folder ID: "default" or specify a Google Drive folder ID  
   - Add Google Docs OAuth2 credentials (OAuth2 setup required)

10. **Create Google Docs Node ("Update Google Docs")**  
    - Connect from "Create a Google Doc"  
    - Operation: Update  
    - Document URL: `={{ $json.id }}` (gets document ID from creation)  
    - Actions: Insert text with expression `={{ $('Extract Human Readable Data').item.json.text }}`  
    - Use same Google Docs OAuth2 credentials

11. **Connect "Update Google Docs" output back to "Loop Over Items"**  
    - This loops processing for remaining batch items

12. **Add Sticky Notes for Documentation (Optional)**  
    - Add notes near groups of nodes to describe steps (input setup, credentials setup, data extraction, AI processing, export)

---

### 5. General Notes & Resources

| Note Content                                                                                               | Context or Link                                                   |
|------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------|
| Workflow extracts Hacker News data, enriches it with Google Gemini 2.0 Flash AI, and exports to Google Docs | Main project description                                         |
| Uses Google Palm API credentials for AI model integration                                                  | Google Gemini integration requires Google Palm API credentials  |
| Google Docs OAuth2 credentials needed for document creation and updates                                    | Google Docs API authentication setup                             |
| Retry on fail enabled on HTTP requests and AI processing ensures better robustness                         | Network or API errors mitigation                                 |
| Hacker News node returns highlighted URL under `_highlightResult.url.value`; fallback logic may be needed | Potential edge case for missing highlights                       |

---

**Disclaimer:** The provided text originates exclusively from an automated n8n workflow. It complies strictly with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly accessible.