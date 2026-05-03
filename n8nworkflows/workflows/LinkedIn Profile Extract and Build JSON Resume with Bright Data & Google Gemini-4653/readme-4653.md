LinkedIn Profile Extract and Build JSON Resume with Bright Data & Google Gemini

https://n8nworkflows.xyz/workflows/linkedin-profile-extract-and-build-json-resume-with-bright-data---google-gemini-4653


# LinkedIn Profile Extract and Build JSON Resume with Bright Data & Google Gemini

### 1. Workflow Overview

This workflow automates the extraction of LinkedIn profile data and transforms it into a structured JSON resume, including a detailed skill extraction. It leverages Bright Data’s web scraping API to fetch the LinkedIn profile content in markdown format and uses Google Gemini Large Language Models (LLMs) for advanced natural language processing tasks to convert, extract, and structure the data.

The workflow is logically divided into the following blocks:

- **1.1 Input Setup:** Manual trigger and initialization of key parameters such as LinkedIn URL, Bright Data zone, and webhook URL.
- **1.2 Data Acquisition:** Execution of a Bright Data web request to scrape LinkedIn profile data.
- **1.3 Markdown Processing:** Conversion of markdown data into clean textual content using Google Gemini LLM.
- **1.4 Resume Extraction:** Using LLMs to extract a structured JSON resume from the textual data.
- **1.5 Skill Extraction:** Mining textual data to extract detailed skill information with descriptions.
- **1.6 Data Persistence & Notification:** Writing structured data and skills to disk and sending a webhook notification with the resume JSON.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Setup

- **Overview:** Receives manual trigger input and sets essential parameters for the LinkedIn profile scraping and workflow control.
- **Nodes Involved:**  
  - When clicking ‘Test workflow’  
  - Set URL and Bright Data Zone  
- **Node Details:**

  - **When clicking ‘Test workflow’**  
    - Type: Manual Trigger  
    - Role: Starting point for manual execution of the workflow.  
    - Configuration: No parameters; triggers workflow on demand.  
    - Inputs: None  
    - Outputs: Connects to "Set URL and Bright Data Zone"  
    - Potential Failures: None typically, except user forgetting to trigger.

  - **Set URL and Bright Data Zone**  
    - Type: Set  
    - Role: Sets workflow variables: LinkedIn profile URL, Bright Data zone, and webhook notification URL.  
    - Configuration:  
      - `url`: LinkedIn profile URL (default example: "https://www.linkedin.com/in/ranjan-dailata")  
      - `zone`: Bright Data scraping zone name (default: "web_unlocker1")  
      - `webhook_notification_url`: URL to notify after data extraction (default example provided).  
    - Inputs: From manual trigger  
    - Outputs: To "Perform Bright Data Web Request"  
    - Edge Cases: Incorrect or malformed URL, wrong Bright Data zone causing request failures.

#### 1.2 Data Acquisition

- **Overview:** Sends a POST request to Bright Data API to scrape the LinkedIn profile page and receive raw markdown data.
- **Nodes Involved:**  
  - Perform Bright Data Web Request  
- **Node Details:**

  - **Perform Bright Data Web Request**  
    - Type: HTTP Request  
    - Role: Calls Bright Data’s API with credentials to scrape the target URL.  
    - Configuration:  
      - Method: POST  
      - URL: https://api.brightdata.com/request  
      - Body parameters include `zone`, `url`, `format: raw`, and `data_format: markdown`  
      - Authentication: HTTP Header Auth with Bright Data API key credential  
    - Inputs: From "Set URL and Bright Data Zone"  
    - Outputs: Raw markdown data to "Markdown to Textual Data Extractor"  
    - Failure Modes: Authentication errors, rate limits, incorrect zone or URL, Bright Data API downtime.

#### 1.3 Markdown Processing

- **Overview:** Converts the raw markdown scrape into clean textual data for further processing.  
- **Nodes Involved:**  
  - Markdown to Textual Data Extractor  
  - Google Gemini Chat Model for Markdown to Textual  
- **Node Details:**

  - **Markdown to Textual Data Extractor**  
    - Type: LangChain Chain LLM  
    - Role: Parses and cleans the markdown input into plain textual data without links, scripts, or style.  
    - Configuration:  
      - Prompt instructs the LLM as a markdown expert to output textual data only.  
      - Input uses the scraped markdown (`{{ $json.data }}`).  
      - Retry enabled on failure.  
    - Inputs: Raw markdown from Bright Data request  
    - Outputs: Clean textual data to both "Skill Extractor" and "JSON Resume Extractor"  
    - Failure Modes: LLM API failure, prompt misinterpretation, malformed markdown input.

  - **Google Gemini Chat Model for Markdown to Textual**  
    - Type: LangChain Google Gemini Chat Model  
    - Role: LLM backend for the markdown extractor node.  
    - Configuration: Model `models/gemini-2.0-flash-exp` with Google PaLM API credentials.  
    - Inputs: Connected internally by LangChain node.  
    - Outputs: Processed textual output to "Markdown to Textual Data Extractor."  
    - Failure Modes: API quota exhaustion, network errors, model response errors.

#### 1.4 Resume Extraction

- **Overview:** Extracts a structured, detailed JSON resume from the clean textual data using an LLM-based information extractor with a strict JSON schema.  
- **Nodes Involved:**  
  - JSON Resume Extractor  
  - Google Gemini Chat Model  
  - Code  
- **Node Details:**

  - **JSON Resume Extractor**  
    - Type: LangChain Information Extractor  
    - Role: Extracts comprehensive resume data (basics, work, education, awards, skills, projects, etc.) into JSON format based on a detailed JSON schema.  
    - Configuration:  
      - Text input: `Extract the resume in JSON format.\n{{ $json.text }}`  
      - Schema: Custom JSON schema for a full resume conforming to JSON Resume standard, including nested objects and arrays.  
      - Retry enabled.  
    - Inputs: Clean textual data from "Markdown to Textual Data Extractor"  
    - Outputs: JSON resume object to "Code" node  
    - Failure Modes: Schema validation errors, LLM extraction inaccuracies, incomplete data.

  - **Google Gemini Chat Model**  
    - Type: LangChain Google Gemini Chat Model  
    - Role: Backend LLM for JSON Resume Extractor.  
    - Configuration: Model `models/gemini-2.0-flash-exp` with Google PaLM credentials.  
    - Inputs: Internal to extractor node.  
    - Outputs: JSON resume data.

  - **Code**  
    - Type: Code (JavaScript)  
    - Role: Passes through the JSON resume extracted output without modification.  
    - Configuration: Returns the first input JSON’s output property.  
    - Inputs: JSON resume from extractor  
    - Outputs: To "Initiate a Webhook Notification for Structured Data" and "Create a binary data for Structured Data Extract"  
    - Failure Modes: Minimal; only code errors in JS.

#### 1.5 Skill Extraction

- **Overview:** Mines the textual data to extract an array of skills with associated descriptions, using an LLM extractor and Google Gemini model.  
- **Nodes Involved:**  
  - Skill Extractor  
  - Google Gemini Chat Model for Skill Extractor  
  - Create a binary data for Structured Skill Extract  
  - Write the structured skills content to disk  
- **Node Details:**

  - **Skill Extractor**  
    - Type: LangChain Information Extractor  
    - Role: Extracts an array of skills and descriptions from the resume text.  
    - Configuration:  
      - Input text: `Perform Data Mining and extract the skills from the provided resume\n\n{{ $json.text }}`  
      - Output schema: Array of objects with `skill` and `desc` string properties.  
      - Retry enabled.  
    - Inputs: Clean textual data from "Markdown to Textual Data Extractor"  
    - Outputs: Skills JSON to "Create a binary data for Structured Skill Extract"  
    - Failure Modes: LLM extraction errors, missing skill data.

  - **Google Gemini Chat Model for Skill Extractor**  
    - Type: LangChain Google Gemini Chat Model  
    - Role: Backend LLM for the Skill Extractor node.  
    - Configuration: Model `models/gemini-2.0-flash-exp` with Google PaLM credentials.  
    - Inputs: Internal to Skill Extractor node.  
    - Outputs: Skills JSON.

  - **Create a binary data for Structured Skill Extract**  
    - Type: Function  
    - Role: Converts the extracted skills JSON into base64-encoded binary data for file writing.  
    - Configuration: Uses Node.js Buffer to encode JSON string.  
    - Inputs: Skills JSON  
    - Outputs: Binary data to "Write the structured skills content to disk"  

  - **Write the structured skills content to disk**  
    - Type: Read/Write File  
    - Role: Writes skills JSON binary data to local disk file `d:\Resume_Skills.json`.  
    - Configuration: Operation `write`, filename set to absolute Windows path.  
    - Inputs: Binary data from function node.  
    - Outputs: None  
    - Failure Modes: File system permission errors, invalid path, disk full.

#### 1.6 Data Persistence & Notification

- **Overview:** Writes the structured JSON resume to disk and sends a webhook notification with the resume data.  
- **Nodes Involved:**  
  - Create a binary data for Structured Data Extract  
  - Write the structured content to disk  
  - Initiate a Webhook Notification for Structured Data  
- **Node Details:**

  - **Create a binary data for Structured Data Extract**  
    - Type: Function  
    - Role: Converts the JSON resume output into base64-encoded binary for file storage.  
    - Configuration: Similar to skill extract function, uses Buffer to encode JSON string.  
    - Inputs: JSON resume from "Code" node  
    - Outputs: Binary data to "Write the structured content to disk"  

  - **Write the structured content to disk**  
    - Type: Read/Write File  
    - Role: Writes the JSON resume binary data to file `d:\Json_Resume.json`.  
    - Configuration: Operation `write`, absolute Windows path.  
    - Inputs: Binary data from function node.  
    - Outputs: None  
    - Failure Modes: File system errors as above.

  - **Initiate a Webhook Notification for Structured Data**  
    - Type: HTTP Request  
    - Role: Sends a POST request to the configured webhook URL with the JSON resume as payload.  
    - Configuration:  
      - URL from the `webhook_notification_url` parameter set initially.  
      - Body contains `json_resume` property with stringified JSON resume.  
      - Sends body as form parameters.  
    - Inputs: JSON resume from "Code" node  
    - Outputs: None  
    - Failure Modes: Network failures, invalid webhook URL, webhook unavailability.

---

### 3. Summary Table

| Node Name                               | Node Type                             | Functional Role                               | Input Node(s)                       | Output Node(s)                                       | Sticky Note                                                                                      |
|----------------------------------------|-------------------------------------|-----------------------------------------------|-----------------------------------|-----------------------------------------------------|------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’           | Manual Trigger                      | Workflow manual start                         | None                              | Set URL and Bright Data Zone                         | ## Note\n\nDeals with the LinkedIn profile data extraction by utilizing the Bright Data and Google Gemini LLM for transforming the profile into a structured JSON resume with the structured skill extraction.\n\n**Please make sure to set the input fields node with the LinkedIn profile URL, Bright Data zone name, Webhook notification URL** |
| Set URL and Bright Data Zone            | Set                                | Sets URL, Bright Data zone, webhook URL       | When clicking ‘Test workflow’     | Perform Bright Data Web Request                      |                                                                                                |
| Perform Bright Data Web Request          | HTTP Request                       | Calls Bright Data API to scrape LinkedIn      | Set URL and Bright Data Zone      | Markdown to Textual Data Extractor                   |                                                                                                |
| Markdown to Textual Data Extractor       | LangChain Chain LLM                | Converts markdown to clean text                | Perform Bright Data Web Request   | Skill Extractor, JSON Resume Extractor               | ## LLM Usages\n\nGoogle Gemini LLM is being utilized for the structured data extraction handling. |
| Google Gemini Chat Model for Markdown to Textual | LangChain Google Gemini Chat Model | LLM backend for markdown to text extraction   | Internal to Markdown to Textual Data Extractor | Markdown to Textual Data Extractor                    |                                                                                                |
| Skill Extractor                         | LangChain Information Extractor     | Extracts skills and descriptions from text    | Markdown to Textual Data Extractor | Create a binary data for Structured Skill Extract    |                                                                                                |
| Google Gemini Chat Model for Skill Extractor | LangChain Google Gemini Chat Model | LLM backend for skill extraction               | Internal to Skill Extractor       | Skill Extractor                                      |                                                                                                |
| Create a binary data for Structured Skill Extract | Function                         | Encodes skills JSON to base64 binary           | Skill Extractor                  | Write the structured skills content to disk          |                                                                                                |
| Write the structured skills content to disk | Read/Write File                   | Writes skills JSON file to disk                 | Create a binary data for Structured Skill Extract | None                                                |                                                                                                |
| JSON Resume Extractor                   | LangChain Information Extractor     | Extracts structured JSON resume from text     | Markdown to Textual Data Extractor | Code                                                |                                                                                                |
| Google Gemini Chat Model                | LangChain Google Gemini Chat Model  | LLM backend for JSON resume extraction         | Internal to JSON Resume Extractor | JSON Resume Extractor                                |                                                                                                |
| Code                                   | Code                              | Passes JSON resume output                       | JSON Resume Extractor            | Initiate a Webhook Notification for Structured Data, Create a binary data for Structured Data Extract |                                                                                                |
| Create a binary data for Structured Data Extract | Function                         | Encodes JSON resume to base64 binary            | Code                            | Write the structured content to disk                 |                                                                                                |
| Write the structured content to disk    | Read/Write File                   | Writes JSON resume file to disk                  | Create a binary data for Structured Data Extract | None                                                |                                                                                                |
| Initiate a Webhook Notification for Structured Data | HTTP Request                    | Sends JSON resume to configured webhook URL    | Code                            | None                                                |                                                                                                |
| Sticky Note1                           | Sticky Note                      | Workflow note on LinkedIn data extraction       | None                            | None                                                | ## Note\n\nDeals with the LinkedIn profile data extraction by utilizing the Bright Data and Google Gemini LLM for transforming the profile into a structured JSON resume with the structured skill extraction.\n\n**Please make sure to set the input fields node with the LinkedIn profile URL, Bright Data zone name, Webhook notification URL** |
| Sticky Note4                           | Sticky Note                      | Notes Google Gemini LLM usage                    | None                            | None                                                | ## LLM Usages\n\nGoogle Gemini LLM is being utilized for the structured data extraction handling. |
| Sticky Note5                           | Sticky Note                      | Branding logo for Bright Data                     | None                            | None                                                | ## Logo\n\n\n![logo](https://images.seeklogo.com/logo-png/43/1/brightdata-logo-png_seeklogo-439974.png) |
| Sticky Note                           | Sticky Note                      | Title note for structured data extract block     | None                            | None                                                | ## Structured Data Extract using LLM                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Name: "When clicking ‘Test workflow’"  
   - No special configuration.

2. **Add a Set Node**  
   - Type: Set  
   - Name: "Set URL and Bright Data Zone"  
   - Assign variables:  
     - `url` (string): LinkedIn profile URL (e.g., "https://www.linkedin.com/in/ranjan-dailata")  
     - `zone` (string): Bright Data zone name (e.g., "web_unlocker1")  
     - `webhook_notification_url` (string): URL for webhook notification (e.g., "https://webhook.site/your-url")  
   - Connect "When clicking ‘Test workflow’" → "Set URL and Bright Data Zone"

3. **Add HTTP Request Node to Call Bright Data**  
   - Type: HTTP Request  
   - Name: "Perform Bright Data Web Request"  
   - Set method to POST and URL to `https://api.brightdata.com/request`  
   - Authentication: Use HTTP Header Auth credential with Bright Data API key  
   - Body parameters (form):  
     - `zone`: `={{ $json.zone }}`  
     - `url`: `={{ $json.url }}`  
     - `format`: "raw"  
     - `data_format`: "markdown"  
   - Connect "Set URL and Bright Data Zone" → "Perform Bright Data Web Request"

4. **Add LangChain Chain LLM Node for Markdown to Text Conversion**  
   - Type: LangChain Chain LLM  
   - Name: "Markdown to Textual Data Extractor"  
   - Prompt: "You need to analyze the below markdown and convert to textual data. Please do not output with your own thoughts. Make sure to output with textual data only with no links, scripts, css etc.\n\n{{ $json.data }}"  
   - Add system message: "You are a markdown expert"  
   - Retry on fail enabled  
   - Connect "Perform Bright Data Web Request" → "Markdown to Textual Data Extractor"

5. **Add Google Gemini Chat Model Node for Markdown Processing**  
   - Type: LangChain Google Gemini Chat Model  
   - Name: "Google Gemini Chat Model for Markdown to Textual"  
   - Model: "models/gemini-2.0-flash-exp"  
   - Credentials: Google PaLM API  
   - Connect this node as the language model backend for "Markdown to Textual Data Extractor" (via ai_languageModel input)

6. **Add LangChain Information Extractor Node for Resume Extraction**  
   - Type: LangChain Information Extractor  
   - Name: "JSON Resume Extractor"  
   - Text: "Extract the resume in JSON format.\n {{ $json.text }}"  
   - Schema: Provide the detailed JSON Resume schema as defined in the original workflow (full JSON Resume draft-07 schema)  
   - Retry on fail enabled  
   - Connect "Markdown to Textual Data Extractor" → "JSON Resume Extractor"

7. **Add Google Gemini Chat Model Node for Resume Extraction**  
   - Type: LangChain Google Gemini Chat Model  
   - Name: "Google Gemini Chat Model"  
   - Model: "models/gemini-2.0-flash-exp"  
   - Credentials: Google PaLM API  
   - Connect as language model backend for "JSON Resume Extractor"

8. **Add LangChain Information Extractor Node for Skill Extraction**  
   - Type: LangChain Information Extractor  
   - Name: "Skill Extractor"  
   - Text: "Perform Data Mining and extract the skills from the provided resume\n\n {{ $json.text }}"  
   - Schema: Array of objects with properties `skill` (string) and `desc` (string)  
   - Retry on fail enabled  
   - Connect "Markdown to Textual Data Extractor" → "Skill Extractor"

9. **Add Google Gemini Chat Model Node for Skill Extraction**  
   - Type: LangChain Google Gemini Chat Model  
   - Name: "Google Gemini Chat Model for Skill Extractor"  
   - Model: "models/gemini-2.0-flash-exp"  
   - Credentials: Google PaLM API  
   - Connect as language model backend for "Skill Extractor"

10. **Add Function Node to Create Binary Data for Skill Extract**  
    - Type: Function  
    - Name: "Create a binary data for Structured Skill Extract"  
    - Code:  
      ```javascript
      items[0].binary = {
        data: {
          data: Buffer.from(JSON.stringify(items[0].json, null, 2)).toString('base64'),
        },
      };
      return items;
      ```  
    - Connect "Skill Extractor" → "Create a binary data for Structured Skill Extract"

11. **Add Read/Write File Node to Save Skills JSON**  
    - Type: Read/Write File  
    - Name: "Write the structured skills content to disk"  
    - Operation: Write  
    - File name: `d:\Resume_Skills.json` (adjust path as needed)  
    - Connect "Create a binary data for Structured Skill Extract" → "Write the structured skills content to disk"

12. **Add Code Node to Pass JSON Resume Output**  
    - Type: Code  
    - Name: "Code"  
    - Code:  
      ```javascript
      return $input.first().json.output;
      ```  
    - Connect "JSON Resume Extractor" → "Code"

13. **Add Function Node to Create Binary Data for JSON Resume**  
    - Type: Function  
    - Name: "Create a binary data for Structured Data Extract"  
    - Code:  
      ```javascript
      items[0].binary = {
        data: {
          data: Buffer.from(JSON.stringify(items[0].json, null, 2)).toString('base64'),
        },
      };
      return items;
      ```  
    - Connect "Code" → "Create a binary data for Structured Data Extract"

14. **Add Read/Write File Node to Save JSON Resume**  
    - Type: Read/Write File  
    - Name: "Write the structured content to disk"  
    - Operation: Write  
    - File name: `d:\Json_Resume.json` (adjust path as needed)  
    - Connect "Create a binary data for Structured Data Extract" → "Write the structured content to disk"

15. **Add HTTP Request Node to Send Webhook Notification**  
    - Type: HTTP Request  
    - Name: "Initiate a Webhook Notification for Structured Data"  
    - Method: POST  
    - URL: `={{ $json.webhook_notification_url }}` (from initial Set node)  
    - Body Parameters:  
      - `json_resume`: `={{ $('JSON Resume Extractor').item.json.output.toJsonString() }}`  
    - Connect "Code" → "Initiate a Webhook Notification for Structured Data"

---

### 5. General Notes & Resources

| Note Content                                                                                                                        | Context or Link                                                                                  |
|------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Workflow uses Google Gemini LLMs extensively for structured data extraction tasks.                                                  | See Sticky Note4: "Google Gemini LLM is being utilized for the structured data extraction handling." |
| Bright Data is used as the scraping service for accessing LinkedIn profile data in markdown format.                                | Branding logo linked in Sticky Note5: ![Bright Data Logo](https://images.seeklogo.com/logo-png/43/1/brightdata-logo-png_seeklogo-439974.png) |
| Users must configure LinkedIn profile URL, Bright Data zone, and webhook notification URL before running the workflow.            | See Sticky Note1 for important user instructions.                                              |
| JSON Resume extraction conforms to JSON Resume schema draft-07 with extensive properties including work, education, skills, etc. | Schema detailed inside "JSON Resume Extractor" node configuration in the workflow.             |

---

**Disclaimer:** The provided text is exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with content policies and contains no illegal, offensive, or protected material. All data handled is legal and public.