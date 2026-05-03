Realtime Job Description & Salary Extraction using Bright Data MCP & OpenAI 4o mini

https://n8nworkflows.xyz/workflows/realtime-job-description---salary-extraction-using-bright-data-mcp---openai-4o-mini-4829


# Realtime Job Description & Salary Extraction using Bright Data MCP & OpenAI 4o mini

### 1. Workflow Overview

This workflow automates the real-time extraction of detailed job descriptions and salary information from online job postings by leveraging Bright Data's MCP Client tools and OpenAI’s GPT-4o mini language model. It targets HR professionals, recruiters, and data analysts who need to gather structured job-related data from web sources such as Indeed and Google search results. The workflow is logically divided into the following blocks:

- **1.1 Input Initialization:** Manual trigger and input field setup for URLs and job roles.
- **1.2 Data Extraction via Bright Data MCP Client:** Scraping job posting content as markdown and performing salary-related search queries.
- **1.3 AI-Powered Structured Data Extraction:** Using OpenAI GPT-4o mini models for converting raw scraped data into structured job description and salary information.
- **1.4 Data Aggregation and Output Processing:** Merging AI responses, creating binary data, writing to files, sending webhook notifications, and updating Google Sheets.
- **1.5 Supporting and Informational Nodes:** Sticky notes and disclaimers for user guidance and branding.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Initialization

- **Overview:**  
  This block handles the manual start of the workflow and the setup of key input parameters such as the job posting URL, job role description, and webhook notification URL.

- **Nodes Involved:**  
  - When clicking ‘Test workflow’ (Manual Trigger)  
  - Bright Data MCP Client List Tools  
  - Set input fields

- **Node Details:**

  1. **When clicking ‘Test workflow’**  
     - *Type & Role:* Manual Trigger — initiates workflow execution manually.  
     - *Configuration:* No parameters; triggered by user interaction.  
     - *Connections:* Outputs to Bright Data MCP Client List Tools.  
     - *Failure Modes:* None typical; user must trigger manually.

  2. **Bright Data MCP Client List Tools**  
     - *Type & Role:* MCP Client node to list available tools, ensuring connectivity and tool readiness.  
     - *Configuration:* Uses MCP Client API credential “MCP Client (STDIO) account”.  
     - *Connections:* Outputs to Set input fields node.  
     - *Failure Modes:* API authentication failure, network issues.

  3. **Set input fields**  
     - *Type & Role:* Set node to define input variables for the workflow.  
     - *Configuration:* Sets three fields:  
       - `job_search_url`: Example Indeed job posting URL.  
       - `job_role`: Text describing the job role and location.  
       - `webhook_notification_url`: URL to send processed results asynchronously.  
     - *Connections:* Outputs to two MCP Client nodes for job data and salary extraction.  
     - *Failure Modes:* Misconfigured or empty input values could cause downstream failures.

---

#### 2.2 Data Extraction via Bright Data MCP Client

- **Overview:**  
  This block uses Bright Data MCP Client tools to scrape the job posting content as markdown and to perform a salary-related Google search based on the job role.

- **Nodes Involved:**  
  - MCP Client for Job Data Extract with Markdown  
  - MCP Client for Salary Data Extraction

- **Node Details:**

  1. **MCP Client for Job Data Extract with Markdown**  
     - *Type & Role:* MCP Client node executing "scrape_as_markdown" tool to scrape the job posting URL.  
     - *Configuration:* Tool parameters include the URL from `job_search_url`.  
     - *Credentials:* MCP Client API credential.  
     - *Connections:* Outputs scraped content to Job Description Extractor.  
     - *Failure Modes:* Scraping errors, invalid URL, network timeout, MCP tool failure.

  2. **MCP Client for Salary Data Extraction**  
     - *Type & Role:* MCP Client node executing "search_engine" tool to query Google with the job role text.  
     - *Configuration:* Query parameter uses `job_role`; search engine set to Google.  
     - *Credentials:* MCP Client API credential.  
     - *Connections:* Outputs to Salary Information Extractor.  
     - *Failure Modes:* Search API failures, query limits, network issues.

---

#### 2.3 AI-Powered Structured Data Extraction

- **Overview:**  
  This block uses OpenAI GPT-4o mini language models via Langchain nodes to convert raw scraped markdown or search results into structured JSON data representing job descriptions and salary details.

- **Nodes Involved:**  
  - OpenAI Chat Model for Job Desc Extract  
  - Job Description Extractor  
  - OpenAI Chat Model for Salary Info Extract  
  - Salary Information Extractor

- **Node Details:**

  1. **OpenAI Chat Model for Job Desc Extract**  
     - *Type & Role:* Langchain OpenAI Chat model node using GPT-4o mini for job description extraction.  
     - *Configuration:* Model set to "gpt-4o-mini".  
     - *Credentials:* OpenAI API credential.  
     - *Connections:* AI output feeds into Job Description Extractor (ai_languageModel input).  
     - *Failure Modes:* API quota limits, timeout, invalid responses.

  2. **Job Description Extractor**  
     - *Type & Role:* Langchain Information Extractor node that parses markdown text to extract detailed job description.  
     - *Configuration:* Extracts textual data from the scraped markdown content.  
     - *Inputs:* Receives raw markdown from MCP Client and processed data from OpenAI model.  
     - *Connections:* Outputs to Merge the response node.  
     - *Failure Modes:* Parsing errors, unexpected markdown formats.

  3. **OpenAI Chat Model for Salary Info Extract**  
     - *Type & Role:* Langchain OpenAI Chat model node using GPT-4o mini for salary extraction.  
     - *Configuration:* Model "gpt-4o-mini".  
     - *Credentials:* OpenAI API credential.  
     - *Connections:* AI output feeds into Salary Information Extractor (ai_languageModel input).  
     - *Failure Modes:* Same as other OpenAI nodes.

  4. **Salary Information Extractor**  
     - *Type & Role:* Langchain Information Extractor node that extracts structured salary information based on a detailed JSON schema defining fields like job title, salary range, employment type, bonus, equity, etc.  
     - *Configuration:* Parses textual content from Google search results and AI output to structured data.  
     - *Connections:* Outputs to Merge the response node.  
     - *Failure Modes:* Schema mismatch, incomplete data, parsing errors.

---

#### 2.4 Data Aggregation and Output Processing

- **Overview:**  
  This block merges the extracted job and salary data, aggregates outputs, converts data to binary for storage, writes JSON files locally, sends webhook notifications, and updates Google Sheets with the collected information.

- **Nodes Involved:**  
  - Merge the response  
  - Aggregate  
  - Create a binary data  
  - Write the salary info to disk  
  - Webhook Notification for Job Info  
  - Update Google Sheets

- **Node Details:**

  1. **Merge the response**  
     - *Type & Role:* Merge node combining job description and salary extraction outputs into a unified data stream.  
     - *Connections:* Outputs to Aggregate node.  
     - *Failure Modes:* Data mismatch, empty inputs.

  2. **Aggregate**  
     - *Type & Role:* Aggregate node to combine all outputs into a single aggregated object with field “output”.  
     - *Connections:* Outputs to three parallel downstream nodes: Create a binary data, Webhook Notification, and Google Sheets update.  
     - *Failure Modes:* Large payloads, memory constraints.

  3. **Create a binary data**  
     - *Type & Role:* Function node converting aggregated JSON output to a base64 encoded binary buffer for file writing.  
     - *Key Code:* Uses Buffer to encode JSON string.  
     - *Connections:* Outputs to Write the salary info to disk.  
     - *Failure Modes:* Encoding errors, node execution errors.

  4. **Write the salary info to disk**  
     - *Type & Role:* Read/Write File node writing the base64-encoded JSON data to a local file path with timestamped filename.  
     - *Configuration:* Writes to `d:\JobDesc-SalaryInfo-{timestamp}.json`.  
     - *Failure Modes:* File system permissions, path invalid, disk full.

  5. **Webhook Notification for Job Info**  
     - *Type & Role:* HTTP Request node sending the aggregated salary info JSON to an external webhook URL provided in input fields.  
     - *Configuration:* Sends POST request with response JSON in body parameter `response`.  
     - *Failure Modes:* Network failures, webhook URL incorrect or unreachable.

  6. **Update Google Sheets**  
     - *Type & Role:* Google Sheets node appending or updating a sheet with the aggregated JSON output string.  
     - *Configuration:* Uses OAuth2 credentials linked to a specific Google Sheet (`documentId=10gAihQMT8-h8Mpehe9j-xxN4oTTpg8qwToI-I-Eauew`) and sheet `gid=0` (Sheet1).  
     - *Failure Modes:* Authentication failure, API quota, invalid sheet ID.

---

#### 2.5 Supporting and Informational Nodes

- **Overview:**  
  Sticky notes provide disclaimers, usage notes, branding, and instructions for users.

- **Nodes Involved:**  
  - Sticky Note2  
  - Sticky Note3  
  - Sticky Note4  
  - Sticky Note5

- **Node Details:**

  1. **Sticky Note2**  
     - Content: Disclaimer stating the template requires n8n self-hosted environment due to MCP Client community node.  
     - Position: Near input nodes for visibility.

  2. **Sticky Note3**  
     - Content: Notes that the workflow extracts job descriptions and salary info using Bright Data MCP Client; reminds to set input fields properly.  

  3. **Sticky Note4**  
     - Content: Notes usage of OpenAI 4o mini LLM for structured data extraction.

  4. **Sticky Note5**  
     - Content: Displays Bright Data logo for branding and visual identification.

---

### 3. Summary Table

| Node Name                             | Node Type                                     | Functional Role                                  | Input Node(s)                            | Output Node(s)                                         | Sticky Note                                                                                      |
|-------------------------------------|-----------------------------------------------|-------------------------------------------------|-----------------------------------------|--------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’        | Manual Trigger                                | Start workflow manually                          | —                                       | Bright Data MCP Client List Tools                       |                                                                                                 |
| Bright Data MCP Client List Tools    | MCP Client                                    | Lists MCP Client tools, validates connection    | When clicking ‘Test workflow’            | Set input fields                                       |                                                                                                 |
| Set input fields                    | Set                                           | Defines job URL, role, and webhook URL inputs   | Bright Data MCP Client List Tools         | MCP Client for Job Data Extract with Markdown, MCP Client for Salary Data Extraction | Sticky Note3: Reminder to set input fields                                                      |
| MCP Client for Job Data Extract with Markdown | MCP Client                            | Scrapes job posting content as markdown         | Set input fields                         | Job Description Extractor                              | Sticky Note2: Self-hosted required due to community node                                        |
| MCP Client for Salary Data Extraction | MCP Client                                  | Performs Google search for salary info           | Set input fields                         | Salary Information Extractor                           | Sticky Note2: Self-hosted required due to community node                                        |
| OpenAI Chat Model for Job Desc Extract | Langchain OpenAI Chat Model                | AI model for extracting structured job description | MCP Client for Job Data Extract with Markdown | Job Description Extractor                              | Sticky Note4: Uses OpenAI 4o mini LLM                                                          |
| Job Description Extractor            | Langchain Information Extractor               | Parses job description text from markdown        | MCP Client for Job Data Extract with Markdown, OpenAI Chat Model for Job Desc Extract | Merge the response                                   |                                                                                                 |
| OpenAI Chat Model for Salary Info Extract | Langchain OpenAI Chat Model                | AI model for extracting structured salary info   | MCP Client for Salary Data Extraction      | Salary Information Extractor                           | Sticky Note4: Uses OpenAI 4o mini LLM                                                          |
| Salary Information Extractor         | Langchain Information Extractor               | Parses salary data per defined JSON schema       | MCP Client for Salary Data Extraction, OpenAI Chat Model for Salary Info Extract | Merge the response                                   |                                                                                                 |
| Merge the response                   | Merge                                          | Combines job description and salary data outputs | Job Description Extractor, Salary Information Extractor | Aggregate                                             |                                                                                                 |
| Aggregate                          | Aggregate                                       | Aggregates all outputs into a single object      | Merge the response                        | Create a binary data, Webhook Notification for Job Info, Update Google Sheets |                                                                                                 |
| Create a binary data                | Function                                       | Encodes aggregated output to base64 binary       | Aggregate                                | Write the salary info to disk                          |                                                                                                 |
| Write the salary info to disk       | Read/Write File                                | Writes JSON file locally with timestamp           | Create a binary data                      | —                                                      |                                                                                                 |
| Webhook Notification for Job Info   | HTTP Request                                   | Sends processed data to external webhook URL     | Aggregate                                | —                                                      |                                                                                                 |
| Update Google Sheets                | Google Sheets                                  | Appends or updates Google Sheet with output data | Aggregate                                | —                                                      |                                                                                                 |
| Sticky Note2                       | Sticky Note                                   | Disclaimer about self-hosted requirement          | —                                       | —                                                      | "This template is only available on n8n self-hosted as it's making use of the community node for MCP Client." |
| Sticky Note3                       | Sticky Note                                   | Reminder note for input fields setup              | —                                       | —                                                      | "Deals with the extraction of Job Description and Salary information by leveraging the Bright Data MCP Client. Please make sure to set the input fields." |
| Sticky Note4                       | Sticky Note                                   | Info about LLM usage                              | —                                       | —                                                      | "OpenAI 4o mini LLM is being utilized for the structured data extraction handling."              |
| Sticky Note5                       | Sticky Note                                   | Branding / logo display                            | —                                       | —                                                      | ![Bright Data Logo](https://images.seeklogo.com/logo-png/43/1/brightdata-logo-png_seeklogo-439974.png) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node:**  
   - Add a Manual Trigger node named “When clicking ‘Test workflow’”. No parameters needed.

2. **Add MCP Client List Tools Node:**  
   - Add MCP Client node named “Bright Data MCP Client List Tools”.  
   - Set credentials to your MCP Client API (e.g., “MCP Client (STDIO) account”).  
   - Connect Manual Trigger → MCP Client List Tools.

3. **Add Set Node for Input Fields:**  
   - Add Set node named “Set input fields”.  
   - Configure three string fields:  
     - `job_search_url`: e.g., "https://www.indeed.com/viewjob?jk=075f6b95d1ae92ae"  
     - `job_role`: e.g., "Lead Backend .Net Developer Dallas, TX Hybrid work"  
     - `webhook_notification_url`: e.g., your webhook URL (https://webhook.site/...)  
   - Connect MCP Client List Tools → Set input fields.

4. **Add MCP Client Node for Job Data Extraction:**  
   - Add MCP Client node named “MCP Client for Job Data Extract with Markdown”.  
   - Set toolName to “scrape_as_markdown”.  
   - Set operation to “executeTool”.  
   - Set toolParameters to JSON with URL from `job_search_url` variable:  
     ```json
     { "url": "{{ $json.job_search_url }}" }
     ```  
   - Use same MCP Client API credentials.  
   - Connect Set input fields → MCP Client for Job Data Extract with Markdown.

5. **Add MCP Client Node for Salary Data Extraction:**  
   - Add MCP Client node named “MCP Client for Salary Data Extraction”.  
   - Set toolName to “search_engine”.  
   - Operation: “executeTool”.  
   - Tool parameters:  
     ```json
     {
       "query": "{{ $json.job_role }}",
       "engine": "google"
     }
     ```  
   - Use MCP Client API credentials.  
   - Connect Set input fields → MCP Client for Salary Data Extraction.

6. **Add OpenAI Chat Model Node for Job Description Extraction:**  
   - Add Langchain OpenAI Chat node named “OpenAI Chat Model for Job Desc Extract”.  
   - Set model to “gpt-4o-mini”.  
   - Configure with OpenAI API credentials.  
   - Connect MCP Client for Job Data Extract with Markdown (main output) → AI input of this node.

7. **Add Job Description Extractor Node:**  
   - Add Langchain Information Extractor node named “Job Description Extractor”.  
   - Configure “text” parameter with expression to extract text from MCP Client output:  
     ```
     Extract markdown to textual data.  {{ $json.result.content[0].text }}
     ```  
   - Define attribute “job_description” as detailed text.  
   - Connect MCP Client for Job Data Extract with Markdown main output → Job Description Extractor main input.  
   - Connect OpenAI Chat Model for Job Desc Extract AI output → Job Description Extractor AI input.

8. **Add OpenAI Chat Model Node for Salary Info Extraction:**  
   - Add Langchain OpenAI Chat node named “OpenAI Chat Model for Salary Info Extract”.  
   - Model: “gpt-4o-mini”.  
   - Use OpenAI API credentials.  
   - Connect MCP Client for Salary Data Extraction main output → AI input.

9. **Add Salary Information Extractor Node:**  
   - Add Langchain Information Extractor node named “Salary Information Extractor”.  
   - Configure “text” parameter with:  
     ```
     Extract the salary information from the following content  {{ $json.result.content[0].text }}
     ```  
   - Paste the provided JSON schema defining fields (job_title, salary_min, salary_max, etc.).  
   - Connect MCP Client for Salary Data Extraction main output → Salary Information Extractor main input.  
   - Connect OpenAI Chat Model for Salary Info Extract AI output → Salary Information Extractor AI input.

10. **Add Merge Node:**  
    - Add Merge node named “Merge the response”.  
    - Connect Job Description Extractor output to input 2.  
    - Connect Salary Information Extractor output to input 1.  
    - This merges job description and salary info.

11. **Add Aggregate Node:**  
    - Add Aggregate node named “Aggregate”.  
    - Set to aggregate field “output”.  
    - Connect Merge the response output → Aggregate input.

12. **Add Function Node to Create Binary Data:**  
    - Add Function node named “Create a binary data”.  
    - Paste function code:  
      ```javascript
      items[0].binary = {
        data: {
          data: Buffer.from(JSON.stringify(items[0].json, null, 2)).toString('base64')
        }
      };
      return items;
      ```  
    - Connect Aggregate output → Create a binary data input.

13. **Add Read/Write File Node:**  
    - Add Read/Write File node named “Write the salary info to disk”.  
    - Operation: write.  
    - File Name:  
      ```
      d:\JobDesc-SalaryInfo-{{Date.now()}}.json
      ```  
    - Connect Create a binary data output → Write the salary info to disk input.

14. **Add HTTP Request Node for Webhook Notification:**  
    - Add HTTP Request node named “Webhook Notification for Job Info”.  
    - URL: use expression from Set input fields node webhook URL, e.g.:  
      ```
      {{$node["Set input fields"].json.webhook_notification_url}}
      ```  
    - Method: POST.  
    - Body Parameters: parameter named “response” with value from aggregated JSON output:  
      ```
      {{$json.output.search_response}}
      ```  
    - Connect Aggregate output → Webhook Notification input.

15. **Add Google Sheets Node:**  
    - Add Google Sheets node named “Update Google Sheets”.  
    - Operation: appendOrUpdate.  
    - Document ID: your Google Sheet ID (e.g., "10gAihQMT8-h8Mpehe9j-xxN4oTTpg8qwToI-I-Eauew").  
    - Sheet Name: “gid=0”.  
    - Columns: map field “output” with value `{{$json.output.toJsonString()}}`.  
    - Connect Aggregate output → Update Google Sheets input.  
    - Configure with Google Sheets OAuth2 credentials.

16. **Add Sticky Notes for Documentation:**  
    - Add sticky notes describing disclaimers, usage notes, and branding near relevant nodes as per your preference.

---

### 5. General Notes & Resources

| Note Content                                                                                   | Context or Link                                                                                                  |
|------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| This template requires n8n self-hosted due to the use of the Bright Data MCP Client community node. | See Sticky Note2 for self-hosted environment disclaimer.                                                        |
| OpenAI GPT-4o mini model is used for advanced structured data extraction from unstructured text. | See Sticky Note4 for LLM usage details.                                                                         |
| Bright Data MCP Client is leveraged for scalable scraping and search engine querying.            | Official Bright Data MCP documentation and API references recommended.                                          |
| Example webhook URL used is from webhook.site for testing; replace with production endpoints.   | Input field `webhook_notification_url` should be updated accordingly.                                           |
| Google Sheets integration requires OAuth2 credentials and correct sheet/document IDs.            | Google Sheets API documentation for OAuth2 setup: https://developers.google.com/sheets/api/guides/authorizing      |
| The salary extraction JSON schema covers comprehensive compensation components and job details. | Helps ensure structured output consistent with HR data standards.                                               |
| Logo image used from SeekLogo for Bright Data branding.                                         | [Bright Data Logo](https://images.seeklogo.com/logo-png/43/1/brightdata-logo-png_seeklogo-439974.png)            |

---

**Disclaimer:**  
The text provided here is a detailed analysis and documentation of an n8n workflow automated using MCP Client and OpenAI nodes. It respects content policies and handles only legal and public data.