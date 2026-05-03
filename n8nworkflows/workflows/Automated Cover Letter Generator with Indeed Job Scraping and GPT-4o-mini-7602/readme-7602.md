Automated Cover Letter Generator with Indeed Job Scraping and GPT-4o-mini

https://n8nworkflows.xyz/workflows/automated-cover-letter-generator-with-indeed-job-scraping-and-gpt-4o-mini-7602


# Automated Cover Letter Generator with Indeed Job Scraping and GPT-4o-mini

### 1. Workflow Overview

This workflow automates the creation of tailored cover letters for job applications leveraging live job data scraped from Indeed and generative AI from OpenAI’s GPT-4o-mini model. It is designed primarily for job seekers or HR professionals aiming to generate personalized cover letters quickly and efficiently based on actual job postings and a provided resume.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Initialization:** Manual trigger to start the workflow and set the search parameters including the job search term and the candidate’s resume.
- **1.2 Job Data Acquisition:** Query the Indeed job listings using Apify’s Indeed Scraper API to fetch relevant job postings.
- **1.3 AI-Powered Cover Letter Generation:** Use the OpenAI GPT-4o-mini model to generate an optimized cover letter customized to the job description and candidate’s resume.
- **1.4 Output Structuring:** Parse and structure the AI-generated output for downstream uses or display.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Initialization

**Overview:**  
This block enables manual initiation of the workflow and sets key input variables: the job search keyword and the full candidate resume text, which is used as context during cover letter generation.

**Nodes Involved:**  
- When clicking ‘Execute workflow’ (Manual Trigger)  
- Set Search Term (Set node)

**Node Details:**

- **When clicking ‘Execute workflow’**  
  - Type: Manual Trigger  
  - Role: Entry point; triggers the entire workflow execution manually.  
  - Configuration: Default, no parameters.  
  - Input connections: None  
  - Output connections: Connects to "Set Search Term" node  
  - Failure modes: None expected, but workflow will not proceed until manually triggered.

- **Set Search Term**  
  - Type: Set node  
  - Role: Defines two key string variables:  
    - `Search`: The job title or keyword for the Indeed search (default "n8n").  
    - `Resume`: A detailed multi-line string containing the candidate’s full resume text.  
  - Configuration: Assignments of the above variables as strings.  
  - Key expressions: Uses static string assignments.  
  - Input connections: From Manual Trigger  
  - Output connections: To "Search Indeed" HTTP Request node  
  - Edge cases: Resume text must be well formatted and appropriate length; excessively long resumes might cause prompt truncation or API limits.

---

#### 2.2 Job Data Acquisition

**Overview:**  
This block performs a live job search by querying Apify’s Indeed Scraper API with the specified search term and location. It fetches up to 10 recent remote jobs matching the term.

**Nodes Involved:**  
- Search Indeed (HTTP Request)

**Node Details:**

- **Search Indeed**  
  - Type: HTTP Request  
  - Role: Makes a POST request to Apify’s Indeed Scraper API endpoint to retrieve job listings.  
  - Configuration:  
    - URL: `https://api.apify.com/v2/acts/misceres~indeed-scraper/run-sync-get-dataset-items`  
    - Method: POST  
    - Body (JSON):  
      - country: "US"  
      - followApplyRedirects: false  
      - location: "remote"  
      - maxItems: 10  
      - parseCompanyDetails: true  
      - position: taken dynamically from `Search` variable set earlier  
      - saveOnlyUniqueItems: true  
    - Authentication: HTTP Query Authentication with Apify API token as query param `token`.  
  - Key expressions: JSON body uses expression `{{ $json.Search }}` from previous node.  
  - Input connections: From "Set Search Term"  
  - Output connections: To "Cover Letter Writer"  
  - Edge cases:  
    - API token invalid or expired: authentication failure.  
    - No jobs found: empty dataset returned; subsequent nodes must handle empty arrays.  
    - API rate limits or downtime.  
  - Version-specific: Requires n8n supporting HTTP Request v4.2 and HTTP Query Authentication.

---

#### 2.3 AI-Powered Cover Letter Generation

**Overview:**  
This block sends the fetched job description and the candidate’s resume to OpenAI’s GPT-4o-mini model, requesting a tailored cover letter with specific formatting instructions.

**Nodes Involved:**  
- Cover Letter Writer (LangChain Agent)  
- OpenAI Chat Model8 (OpenAI Chat Model)  
- Structured Output Parser6 (LangChain Structured Output Parser)

**Node Details:**

- **Cover Letter Writer**  
  - Type: LangChain Agent node  
  - Role: Composes prompt combining job description and resume, sends to AI language model, and expects structured JSON output with a cover letter.  
  - Configuration:  
    - Text prompt:  
      ```
      Job Description: {{ $json.description }} Resume: {{ $('Set Search Term').item.json.Resume }}
      ```  
    - System message: Instructs AI to write an optimized cover letter using only resume content, one paragraph plus bullet points, outputting JSON with key "cover letter".  
    - Prompt type: Defined prompt  
    - Output parser enabled (linked to Structured Output Parser node).  
  - Input connections: From "Search Indeed" (job data)  
  - Output connections: To none (end of workflow)  
  - Sub-node connections: Receives language model reference from "OpenAI Chat Model8" and output parser from "Structured Output Parser6".  
  - Edge cases:  
    - Prompt length exceeding model limits.  
    - Empty or missing job description or resume fields causing incomplete input.  
    - Model output not conforming to expected JSON schema, causing parser failure.  
  - Version-specific: Requires LangChain nodes v2.2 or higher.

- **OpenAI Chat Model8**  
  - Type: LangChain OpenAI Chat Model  
  - Role: Provides GPT-4o-mini language model backend for the agent.  
  - Configuration: Model selection: "gpt-4o-mini"  
  - Credentials: OpenAI API key configured in n8n credentials.  
  - Input connections: Linked as AI language model in "Cover Letter Writer"  
  - Output connections: Linked back to "Cover Letter Writer"  
  - Edge cases:  
    - API key invalid or insufficient billing/funds.  
    - API request timeout or rate limits.  
    - Model unavailable or deprecated.

- **Structured Output Parser6**  
  - Type: LangChain Structured Output Parser  
  - Role: Parses AI response to extract the "cover letter" JSON field reliably.  
  - Configuration: Example JSON schema specifying expected "cover letter" field.  
  - Input connections: Linked to "Cover Letter Writer" as output parser  
  - Output connections: None (end of processing)  
  - Edge cases:  
    - AI output not matching schema causing parsing errors.  
    - Partial or malformed JSON responses.

---

### 3. Summary Table

| Node Name               | Node Type                          | Functional Role                         | Input Node(s)            | Output Node(s)          | Sticky Note                                                                                  |
|-------------------------|----------------------------------|---------------------------------------|--------------------------|------------------------|----------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger                  | Manual start of workflow               | None                     | Set Search Term          |                                                                                              |
| Set Search Term          | Set                              | Defines search term and resume input  | When clicking ‘Execute workflow’ | Search Indeed          |                                                                                              |
| Search Indeed            | HTTP Request                     | Fetches job postings from Indeed via Apify scraper | Set Search Term           | Cover Letter Writer      | See “Setup Instructions” and “Set Up Apify Connection” sticky notes for API key setup        |
| Cover Letter Writer      | LangChain Agent                  | Generates cover letter via OpenAI GPT-4o-mini | Search Indeed             | None                   | See “Set Up OpenAI Connection” sticky notes for API key and billing setup                    |
| OpenAI Chat Model8       | LangChain OpenAI Chat Model      | Provides GPT-4o-mini model backend    | Cover Letter Writer (AI language model) | Cover Letter Writer    | See “Set Up OpenAI Connection” sticky notes                                                 |
| Structured Output Parser6| LangChain Structured Output Parser | Parses AI JSON output for cover letter | Cover Letter Writer       | None                   |                                                                                              |
| Sticky Note25            | Sticky Note                      | Setup instructions and contact info   | None                     | None                   | Setup instructions for OpenAI and Apify API credential configuration                          |
| Sticky Note26            | Sticky Note                      | Workflow description and purpose      | None                     | None                   | Summary of workflow capabilities and integration                                            |
| Sticky Note27            | Sticky Note                      | OpenAI connection setup instructions  | None                     | None                   | OpenAI API key and billing setup                                                           |
| Sticky Note28            | Sticky Note                      | Apify connection setup instructions   | None                     | None                   | Apify API key and scraper setup details                                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**:  
   - Type: Manual Trigger  
   - Purpose: Initiate the workflow manually.

2. **Create Set Node ("Set Search Term")**:  
   - Type: Set  
   - Parameters:  
     - Add string variable `Search` with value e.g. `"n8n"`.  
     - Add string variable `Resume` with the full text of the candidate's resume (multi-line string).  
   - Connect Manual Trigger output to this node’s input.

3. **Create HTTP Request Node ("Search Indeed")**:  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.apify.com/v2/acts/misceres~indeed-scraper/run-sync-get-dataset-items`  
   - Authentication: HTTP Query Auth (create new credential):  
     - Query Key: `token`  
     - Value: your Apify API key from Apify Console.  
   - Body Content Type: JSON  
   - JSON Body:  
     ```json
     {
       "country": "US",
       "followApplyRedirects": false,
       "location": "remote",
       "maxItems": 10,
       "parseCompanyDetails": true,
       "position": "{{$json.Search}}",
       "saveOnlyUniqueItems": true
     }
     ```  
   - Connect Set node output to this HTTP Request node input.

4. **Create LangChain OpenAI Chat Model Node ("OpenAI Chat Model8")**:  
   - Type: LangChain OpenAI Chat Model  
   - Model: Select `gpt-4o-mini`  
   - Credentials: Configure OpenAI API credentials with your API key and ensure billing is enabled.

5. **Create LangChain Structured Output Parser Node ("Structured Output Parser6")**:  
   - Type: LangChain Structured Output Parser  
   - JSON Schema Example:  
     ```json
     {
       "cover letter": "Cover Letter"
     }
     ```

6. **Create LangChain Agent Node ("Cover Letter Writer")**:  
   - Type: LangChain Agent  
   - Parameters:  
     - Text prompt:  
       ```
       Job Description: {{$json.description}} Resume: {{$node["Set Search Term"].json.Resume}}
       ```  
     - System message:  
       ```
       Write an optimized cover letter for the incoming job description. Look to the resume for ideas on what to include. Do not make anything up, only use content from the resume. The cover letter should be one paragraph with bullet points after.

       Output data like this.
       {
         "cover letter": "Cover Letter"
       }
       ```  
     - Prompt type: Define  
     - Enable Output Parser and select the Structured Output Parser node created earlier.  
   - Connect the HTTP Request node ("Search Indeed") output to this node’s input.  
   - Set the LangChain OpenAI Chat Model node as the AI language model for this agent.

7. **Connect the LangChain OpenAI Chat Model node output back to the LangChain Agent as AI model input.**

8. **Ensure all nodes are correctly connected as per the flow:**  
   Manual Trigger → Set Search Term → Search Indeed → Cover Letter Writer (using OpenAI Chat Model and Structured Output Parser).

9. **Save and activate the workflow.**

10. **Test by executing the manual trigger node.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                         | Context or Link                                                                                             |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------|
| ⚙️ Setup Instructions: Configure OpenAI API key with billing and Apify API key with HTTP Query Auth in n8n credentials.                                            | https://platform.openai.com/api-keys, https://console.apify.com/account/integrations                       |
| Workflow automates cover letter generation using live Indeed job data and GPT-4o-mini, producing a paragraph plus bullet points format based solely on resume data. | Sticky Note26 content                                                                                        |
| For help customizing or building similar automations, contact Robert Breen via email or LinkedIn.                                                                  | Email: robert@ynteractive.com, LinkedIn: https://www.linkedin.com/in/robert-breen-29429625/, Website: https://ynteractive.com |

---

**Disclaimer:**  
The provided text is exclusively generated from an automated n8n workflow. It complies strictly with content policies and contains no illegal or offensive material. All data handled is lawful and publicly accessible.