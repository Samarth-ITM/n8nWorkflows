Summarize & Extract Glassdoor Company Info with Google Gemini and Decode

https://n8nworkflows.xyz/workflows/summarize---extract-glassdoor-company-info-with-google-gemini-and-decode-9924


# Summarize & Extract Glassdoor Company Info with Google Gemini and Decode

### 1. Workflow Overview

This workflow automates the extraction, structuring, and summarization of company intelligence data from Glassdoor URLs using the Decodo API and Google Gemini large language models (LLMs). It targets HR professionals, recruiters, and data analysts who want to quickly generate structured, insightful company reports from raw Glassdoor data.

The workflow is logically divided into these blocks:

- **1.1 Input Reception:** Manual trigger and setting of input parameters (Glassdoor company URL and geographic location).  
- **1.2 Raw Data Extraction:** Using Decodo API to retrieve unstructured Glassdoor company information including overview, ratings, reviews, and FAQs.  
- **1.3 Structured Data Extraction:** Leveraging Google Gemini LLM to parse Decodo’s raw data into structured JSON format with company details, ratings breakdown, reviews, and FAQs.  
- **1.4 Summarization:** Using Google Gemini to produce an AI-generated, human-readable summary highlighting company reputation, culture, employee sentiment, strengths, and weaknesses.  
- **1.5 Data Merging and Formatting:** Combining structured data and summary into a single JSON object, formatting it for output.  
- **1.6 File Output and Encoding:** Writing the JSON report to disk and encoding it in Base64 for downstream use such as API uploads or downloads.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block initializes the workflow manually and sets the input parameters, specifically the Glassdoor company URL and geographic region, which are required by the Decodo API.

- **Nodes Involved:**  
  - When clicking ‘Execute workflow’ (Manual Trigger)  
  - Set the Input Fields (Set)

- **Node Details:**

  - **When clicking ‘Execute workflow’**  
    - Type: Manual Trigger  
    - Role: Starts the workflow manually by user interaction.  
    - Configuration: No parameters; default manual trigger.  
    - Inputs: None  
    - Outputs: Connects to "Set the Input Fields" node.  
    - Edge Cases: User must manually click to start; no automation trigger here.

  - **Set the Input Fields**  
    - Type: Set node  
    - Role: Assigns the input variables `company_url` and `geo` used downstream.  
    - Configuration:  
      - `company_url` set to "https://www.glassdoor.co.in/Overview/Working-at-Decode-Technologies-EI_IE4772827.11,30.htm"  
      - `geo` set to "India"  
    - Inputs: Manual Trigger node  
    - Outputs: Connects to the Decodo node.  
    - Edge Cases: Hardcoded values; to reuse, these must be parameterized or dynamically set.

---

#### 2.2 Raw Data Extraction

- **Overview:**  
  This block fetches raw company data from Glassdoor using the Decodo API, based on the input URL and geographic location.

- **Nodes Involved:**  
  - Decodo

- **Node Details:**

  - **Decodo**  
    - Type: Decodo API node (custom integration)  
    - Role: Pulls detailed Glassdoor company data including overview, ratings, reviews, and FAQs.  
    - Configuration: Uses expressions to dynamically set:  
      - `url` = `company_url` from "Set the Input Fields"  
      - `geo` = `geo` from "Set the Input Fields"  
    - Credentials: Decodo API key (OAuth or API token stored in credentials manager)  
    - Inputs: From "Set the Input Fields"  
    - Outputs: Feeds data into both Structured Data Extractor and Summarizer nodes in parallel.  
    - Edge Cases: API failures, rate limits, invalid or inaccessible URL, geo parameter mismatch.  
    - Retry: Enabled on failure to enhance reliability.

---

#### 2.3 Structured Data Extraction

- **Overview:**  
  This block analyzes the raw data from Decodo and extracts structured insights such as company overview, ratings breakdown, review snippets, FAQs, and call-to-action text using Google Gemini LLM and an output parser.

- **Nodes Involved:**  
  - Google Gemini Chat Model for Structured Data Extract  
  - Structured Data Extractor (LangChain Chain LLM)  
  - Structured Output Parser

- **Node Details:**

  - **Google Gemini Chat Model for Structured Data Extract**  
    - Type: LangChain Google Gemini Chat Model  
    - Role: Language model call to parse text into structured data format.  
    - Configuration: Model set to `models/gemini-2.0-flash-exp`  
    - Credentials: Google PaLM API (Google Gemini)  
    - Inputs: Raw data content from Decodo (`data.results[0].content` extracted by expression)  
    - Outputs: Sent to Structured Data Extractor node  
    - Edge Cases: API quota, timeouts, malformed inputs.

  - **Structured Data Extractor**  
    - Type: LangChain Chain LLM with Output Parser  
    - Role: Processes Gemini’s output to extract JSON structured data based on a defined schema.  
    - Configuration:  
      - Text input: "Analyze and extract structured data\n\n{{ $json.data.results[0].content }}"  
      - Uses "Structured Output Parser" node as output parser (ai_outputParser connection)  
    - Inputs: From Gemini Chat Model for Structured Data Extract  
    - Outputs: Feeds output to Merge node (main output)  
    - Edge Cases: Parsing errors if Gemini output deviates from schema.

  - **Structured Output Parser**  
    - Type: LangChain Output Parser Structured  
    - Role: Validates and enforces the JSON schema on LLM’s output.  
    - Configuration: Manual schema defining company details, ratings, reviews, FAQs, and call to action.  
    - Inputs: Connected as ai_outputParser input to Structured Data Extractor  
    - Outputs: Parsed structured JSON to Structured Data Extractor  
    - Edge Cases: Schema mismatch, missing required fields.

---

#### 2.4 Summarization

- **Overview:**  
  Generates a human-readable summary from the raw data highlighting company culture, employee sentiment, strengths, weaknesses, and hiring notes using Google Gemini.

- **Nodes Involved:**  
  - Google Gemini Chat Model for Summarization  
  - Summarizer  
  - Merge

- **Node Details:**

  - **Google Gemini Chat Model for Summarization**  
    - Type: LangChain Google Gemini Chat Model  
    - Role: Generates summary text from Glassdoor raw data.  
    - Configuration: Model `models/gemini-2.0-flash-exp`  
    - Credentials: Google PaLM API  
    - Inputs: Raw Glassdoor data content (`data.results[0].content`) from Decodo node  
    - Outputs: Feeds into Summarizer node

  - **Summarizer**  
    - Type: LangChain Information Extractor  
    - Role: Extracts a detailed summary string from the Gemini output using a manual schema.  
    - Configuration: Input schema expects `detailed_summary` string property.  
    - Inputs: From Gemini Summarization node  
    - Outputs: Connects to the Merge node (merged with structured data)

  - **Merge**  
    - Type: Merge node  
    - Role: Combines outputs from Summarizer and Structured Data Extractor into one unified JSON object.  
    - Inputs: Two inputs - structured data extractor output and summarizer output (main outputs)  
    - Outputs: Feeds into Format the Response node  
    - Edge Cases: Mismatched or missing data causing incomplete merges.

---

#### 2.5 Data Merging and Formatting

- **Overview:**  
  Formats the merged structured and summarized data to extract the first element of the output array for clean JSON output.

- **Nodes Involved:**  
  - Format the Response

- **Node Details:**

  - **Format the Response**  
    - Type: Code node (JavaScript)  
    - Role: Extracts first element of output array for simplified downstream usage.  
    - Configuration: JavaScript returning `$input.first().json.output[0];`  
    - Inputs: From Merge node  
    - Outputs: Feeds into Create a Binary Response node  
    - Edge Cases: Empty input array or unexpected output structure.

---

#### 2.6 File Output and Encoding

- **Overview:**  
  Writes the final JSON report to disk at a dynamic file path based on company name, and encodes the JSON as Base64 for further API upload or download.

- **Nodes Involved:**  
  - Create a Binary Response  
  - Read/Write Files from Disk

- **Node Details:**

  - **Create a Binary Response**  
    - Type: Function node  
    - Role: Converts JSON text to Base64-encoded binary data for file handling or API upload.  
    - Configuration: JavaScript uses `Buffer` to encode JSON string with indentation.  
    - Inputs: From Format the Response node  
    - Outputs: To Read/Write Files from Disk node  
    - Edge Cases: Encoding errors, empty data.

  - **Read/Write Files from Disk**  
    - Type: Read/Write File node  
    - Role: Saves the Base64-encoded JSON data to disk with a dynamic filename.  
    - Configuration:  
      - Operation: Write  
      - Filename: `C:\{{ $json.overview.companyName }}.json` (dynamic company name)  
      - Data Property: `data` (binary Base64 content)  
    - Inputs: From Create a Binary Response node  
    - Edge Cases: File system permissions, invalid filename characters, disk full.

---

### 3. Summary Table

| Node Name                             | Node Type                           | Functional Role                            | Input Node(s)                      | Output Node(s)                   | Sticky Note                                                                                       |
|-------------------------------------|-----------------------------------|--------------------------------------------|----------------------------------|---------------------------------|-------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’    | Manual Trigger                    | Initiates the workflow manually            | None                             | Set the Input Fields             | ![Logo](https://cdn.brandfetch.io/idIeG9_eXK/w/100/h/100/theme/dark/icon.jpeg?c=1bxid64Mup7aczewSAYMX&t=1756483136894) Google Gemini LLM for the Structured Data Extraction and Summarization Purposes |
| Set the Input Fields                 | Set                               | Defines input variables (company_url, geo) | When clicking ‘Execute workflow’ | Decodo                          | See Sticky Note2 below                                                                         |
| Decodo                              | Decodo API                       | Extracts raw Glassdoor data                 | Set the Input Fields             | Structured Data Extractor, Summarizer | See Sticky Note2 below                                                                         |
| Google Gemini Chat Model for Structured Data Extract | LangChain LM Chat Google Gemini | Parses raw data into structured JSON       | Decodo                         | Structured Data Extractor        | Google Gemini LLM Data Extract                                                                 |
| Structured Data Extractor            | LangChain Chain LLM              | Extracts structured data with output parser | Google Gemini Chat Model for Structured Data Extract | Merge                          | Google Gemini LLM Data Extract                                                                 |
| Structured Output Parser             | LangChain Output Parser Structured | Validates and structures LLM output        | Structured Data Extractor (ai_outputParser) | Structured Data Extractor       | Google Gemini LLM Data Extract                                                                 |
| Google Gemini Chat Model for Summarization | LangChain LM Chat Google Gemini | Summarizes raw data into human-readable text | Decodo                         | Summarizer                     | See Sticky Note2 below                                                                         |
| Summarizer                         | LangChain Information Extractor  | Extracts summary string from LLM output    | Google Gemini Chat Model for Summarization | Merge                          | See Sticky Note2 below                                                                         |
| Merge                              | Merge                            | Combines structured data and summary       | Structured Data Extractor, Summarizer | Format the Response             | See Sticky Note2 below                                                                         |
| Format the Response                | Code                             | Simplifies merged output to first array element | Merge                          | Create a Binary Response         |                                                                                                 |
| Create a Binary Response           | Function                         | Encodes JSON text as Base64 binary          | Format the Response              | Read/Write Files from Disk       |                                                                                                 |
| Read/Write Files from Disk          | Read/Write File                  | Writes JSON report to disk with dynamic name | Create a Binary Response         | None                            | See Sticky Note2 below                                                                         |
| Sticky Note1                      | Sticky Note                      | Branding and general info about Google Gemini | None                          | None                           | ![Logo](https://cdn.brandfetch.io/idIeG9_eXK/w/100/h/100/theme/dark/icon.jpeg?c=1bxid64Mup7aczewSAYMX&t=1756483136894) Google Gemini LLM for the Structured Data Extraction and Summarization Purposes |
| Sticky Note                       | Sticky Note                      | Label for the Google Gemini LLM Data Extract block | None                          | None                           | ## Google Gemini LLM Data Extract                                                              |
| Sticky Note2                      | Sticky Note                      | Detailed workflow purpose and flow summary | None                          | None                           | Automate Glassdoor company intelligence extraction using Decodo API and Google Gemini LLM. Generates structured JSON report, summarization, and file export. See full flow steps in note.|

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - No parameters. This node starts the workflow manually.

2. **Create Set Node ("Set the Input Fields")**  
   - Type: Set  
   - Add two string fields:  
     - `company_url` with value `"https://www.glassdoor.co.in/Overview/Working-at-Decode-Technologies-EI_IE4772827.11,30.htm"`  
     - `geo` with value `"India"`  
   - Connect Manual Trigger output to this node.

3. **Create Decodo Node**  
   - Type: Decodo API (custom node, install `@decodo/n8n-nodes-decodo`)  
   - Parameters:  
     - `url` set via expression to `{{$node["Set the Input Fields"].json["company_url"]}}`  
     - `geo` set via expression to `{{$node["Set the Input Fields"].json["geo"]}}`  
   - Credentials: Configure Decodo API credentials (OAuth/API key).  
   - Connect output of Set node to this node.  
   - Enable "Retry on Fail" for robustness.

4. **Create Google Gemini Chat Model Node for Structured Data Extract**  
   - Type: LangChain Google Gemini Chat Model  
   - Parameters:  
     - Model Name: `models/gemini-2.0-flash-exp`  
   - Credentials: Connect Google PaLM API credentials for Google Gemini.  
   - Connect output of Decodo node to this node.

5. **Create Structured Data Extractor Node**  
   - Type: LangChain Chain LLM  
   - Parameters:  
     - Text Input: `Analyze and extract structured data\n\n{{ $json.data.results[0].content }}`  
     - Enable Output Parser with manual schema.  
   - Connect output of Google Gemini Chat Model for Structured Data Extract to this node.  
   - Enable "Retry on Fail".

6. **Create Structured Output Parser Node**  
   - Type: LangChain Output Parser Structured  
   - Parameters: Define JSON schema with:  
     - Array of objects containing:  
       - `text` (string)  
       - `overview` (object with companyName, rating, location, size, type, website)  
       - `keyTakeaways` (string array)  
       - `ratingsBreakdown` (object with keys for culture, diversity, work-life, management, compensation, career)  
       - `reviewSnippets` (array of pros/cons per role)  
       - `faq` (array of question/answer objects)  
       - `callToAction` (string)  
   - Connect this node as the output parser of the Structured Data Extractor node.

7. **Create Google Gemini Chat Model Node for Summarization**  
   - Type: LangChain Google Gemini Chat Model  
   - Parameters:  
     - Model Name: `models/gemini-2.0-flash-exp`  
   - Credentials: Google PaLM API.  
   - Connect output of Decodo node to this node.

8. **Create Summarizer Node**  
   - Type: LangChain Information Extractor  
   - Parameters:  
     - Text: `Summarize the following content  {{ $json.data.results[0].content }}`  
     - Schema: Object with `detailed_summary` string property.  
   - Connect output of Google Gemini Chat Model for Summarization node to this node.

9. **Create Merge Node**  
   - Type: Merge  
   - Connect Structured Data Extractor output as input 1  
   - Connect Summarizer output as input 2  
   - Set to merge data by default (append or merge mode) to combine both results.

10. **Create Format the Response Node**  
    - Type: Code node (JavaScript)  
    - Code: `return $input.first().json.output[0];`  
    - Connect output of Merge node to this node.

11. **Create Create a Binary Response Node**  
    - Type: Function node  
    - Code:  
      ```javascript
      items[0].binary = {
        data: {
          data: Buffer.from(JSON.stringify(items[0].json, null, 2)).toString('base64')
        }
      };
      return items;
      ```  
    - Connect output of Format the Response node to this node.

12. **Create Read/Write Files from Disk Node**  
    - Type: Read/Write File  
    - Parameters:  
      - Operation: Write  
      - File Name: `C:\\{{$json.overview.companyName}}.json` (dynamic file path)  
      - Data Property Name: `data` (binary)  
    - Connect output of Create a Binary Response node to this node.

13. **Add Sticky Notes for Documentation**  
    - Add three sticky notes with branding, workflow purpose, and Google Gemini LLM data extraction notes.  
    - Position them as per user preference.

14. **Activate and Test Workflow**  
    - Verify credentials for Decodo API and Google Gemini (Google PaLM) API.  
    - Test by executing manually and verify output JSON file creation.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                     | Context or Link                                                                                                            |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------|
| Google Gemini LLM is used extensively for both structured data extraction and summarization to leverage advanced AI capabilities in natural language understanding and generation.                                                | Sticky Note1 and Sticky Note                                                                                               |
| The workflow automates transforming unstructured Glassdoor data into structured company intelligence reports, useful for HR and recruitment analytics.                                                                          | Sticky Note2                                                                                                               |
| Decodo API is critical for raw data extraction from Glassdoor; ensure your Decodo API credentials are properly configured with sufficient quota.                                                                                | Workflow credentials                                                                                                       |
| For Google Gemini (PaLM) integration, ensure you have the correct API key and permissions set in Google Cloud Console.                                                                                                         | Google PaLM API setup                                                                                                      |
| File output path is hardcoded to `C:\` drive for Windows systems; modify path accordingly for other environments or OS.                                                                                                        | Read/Write Files from Disk node                                                                                             |
| [Decodo API Documentation](https://decodo.ai/docs) and [Google PaLM API Documentation](https://developers.generativeai.google) provide details on API usage and limits.                                                         | External resources                                                                                                         |

---

**Disclaimer:** The provided text is generated exclusively from an automated workflow created with n8n, a workflow automation tool. It strictly complies with current content policies and contains no illegal, offensive, or protected material. All data handled is legal and publicly available.