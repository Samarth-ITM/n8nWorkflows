TalentFlow AI – Instantly evaluate applicant's GitHub, LinkedIn, using  AI

https://n8nworkflows.xyz/workflows/talentflow-ai---instantly-evaluate-applicant-s-github--linkedin--using--ai-5099


# TalentFlow AI – Instantly evaluate applicant's GitHub, LinkedIn, using  AI

### 1. Workflow Overview

This workflow, titled **TalentFlow AI – Instantly evaluate applicant's GitHub, LinkedIn, using AI**, automates the evaluation of job applicants by analyzing their professional online profiles and resumes. Its core function is to intake new job applications from JotForm, extract relevant usernames and resume data, scrape public data from LinkedIn, GitHub, and LeetCode, and then apply AI-driven assessments to generate structured evaluations. The final output is appended to a Google Sheet for HR review.

The workflow logically divides into these key blocks:

- **1.1 Input Reception and Data Extraction:** Captures new job applications via JotForm webhook, fetches form data, filters required fields, and extracts usernames for LinkedIn, GitHub, and LeetCode as well as the resume file.
- **1.2 Data Validation and Scraping:** Validates the existence of usernames, then scrapes LinkedIn profiles and posts, GitHub repositories, and LeetCode statistics.
- **1.3 Resume Processing:** Converts uploaded PDF resumes to text, fetches the text content, and prepares it for AI evaluation.
- **1.4 AI Evaluation:** Uses OpenRouter-powered language models and LangChain agents to evaluate LinkedIn, GitHub, LeetCode data, and resumes. It applies structured output parsers for consistent AI output.
- **1.5 Aggregation and Output:** Merges all evaluation results, formats the final output, and appends candidate data to a Google Sheet for record-keeping.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Input Reception and Data Extraction

**Overview:**  
This block listens for new job applications via a JotForm webhook, fetches raw submission data, filters required fields, and extracts usernames (LinkedIn, GitHub, LeetCode) along with the resume file link.

**Nodes Involved:**  
- Trigger: New Job Application  
- Fetch JotForm Data  
- Filter Required Fields  
- Extract Usernames from Fields  

**Node Details:**  

- **Trigger: New Job Application**  
  - Type: JotForm Trigger  
  - Role: Entry point; triggers workflow on new form submission.  
  - Config: Uses a webhook ID to listen for submissions.  
  - Inputs: External webhook event.  
  - Outputs: Raw submission data.  
  - Failures: Webhook downtime or missing submissions.

- **Fetch JotForm Data**  
  - Type: HTTP Request  
  - Role: Retrieves detailed submission data from JotForm API.  
  - Config: Uses JotForm API to fetch full form data.  
  - Inputs: Trigger node output (submission ID).  
  - Outputs: Detailed form data JSON.  
  - Failures: API auth errors, rate limits.

- **Filter Required Fields**  
  - Type: Code (JavaScript)  
  - Role: Filters only relevant fields needed for evaluation.  
  - Config: Custom script to parse and extract key fields like usernames and resume URL.  
  - Inputs: Form data JSON.  
  - Outputs: Filtered data containing usernames and resume link.  
  - Failures: Parsing errors if form structure changes.

- **Extract Usernames from Fields**  
  - Type: Code (JavaScript)  
  - Role: Extracts and normalizes usernames for LinkedIn, GitHub, and LeetCode.  
  - Config: Uses regex or string parsing to isolate usernames from URLs or text input.  
  - Inputs: Filtered form data.  
  - Outputs: Cleaned usernames and resume file link for next processing.  
  - Failures: Incorrect username format, missing fields.

---

#### 2.2 Data Validation and Scraping

**Overview:**  
Validates extracted usernames and scrapes public data from LinkedIn profiles/posts, GitHub repositories, and LeetCode statistics via HTTP requests.

**Nodes Involved:**  
- Validate GitHub Username (disabled)  
- Scrape GitHub Data  
- Format Github Data  
- Validate Leetcode Username (disabled)  
- Scrape LeetCode Stats  
- Format LeetCode Data  
- Validate LinkedIn Username (disabled)  
- Scrape LinkedIn Data  
- Scrape LinkedIn Post Data  
- Format LinkedIn Result  
- Format LinkedIn Result1  
- Merge  

**Node Details:**  

- **Validate GitHub Username**  
  - Type: If (disabled)  
  - Role: Conditional check for GitHub username presence.  
  - Outputs: Scrape GitHub Data or No Operation.  
  - Notes: Disabled, so validation bypassed.

- **Scrape GitHub Data**  
  - Type: HTTP Request  
  - Role: Scrapes public GitHub profile data via GitHub API or scraping endpoints.  
  - Inputs: GitHub username.  
  - Outputs: Raw GitHub profile data JSON.  
  - Failures: API rate limits, private profiles, invalid username.

- **Format Github Data**  
  - Type: Code (JavaScript)  
  - Role: Parses and formats GitHub data for AI evaluation.  
  - Inputs: Raw GitHub data.  
  - Outputs: Structured GitHub data.  
  - Failures: Parsing errors if data format changes.

- **Validate Leetcode Username**  
  - Type: If (disabled)  
  - Role: Conditional check for LeetCode username presence.  
  - Outputs: Scrape LeetCode Stats or No Operation.  
  - Notes: Disabled, so validation bypassed.

- **Scrape LeetCode Stats**  
  - Type: HTTP Request  
  - Role: Scrapes public LeetCode user statistics page.  
  - Inputs: LeetCode username.  
  - Outputs: Raw LeetCode stats HTML or JSON.  
  - Failures: Private profiles, invalid usernames, site changes.

- **Format LeetCode Data**  
  - Type: Code (JavaScript)  
  - Role: Extracts relevant LeetCode stats and formats for AI.  
  - Inputs: Raw scrape data.  
  - Outputs: Structured LeetCode stats.  
  - Failures: Parsing errors.

- **Validate LinkedIn Username**  
  - Type: If (disabled)  
  - Role: Conditional check for LinkedIn username presence.  
  - Outputs: Scrape LinkedIn Data/Post Data or No Operation.  
  - Notes: Disabled, so validation bypassed.

- **Scrape LinkedIn Data**  
  - Type: HTTP Request  
  - Role: Scrapes LinkedIn profile data from public pages.  
  - Inputs: LinkedIn username.  
  - Outputs: Raw LinkedIn profile data.  
  - Failures: Captchas, login requirements, private profiles.

- **Scrape LinkedIn Post Data**  
  - Type: HTTP Request  
  - Role: Scrapes LinkedIn posts or activities.  
  - Inputs: LinkedIn username.  
  - Outputs: Raw posts data.  
  - Failures: Same as above.

- **Format LinkedIn Result** / **Format LinkedIn Result1**  
  - Type: Code (JavaScript)  
  - Role: Parses and formats profile and post data separately.  
  - Inputs: Raw scrape data.  
  - Outputs: Structured LinkedIn profile and post data.  
  - Failures: Parsing inconsistencies.

- **Merge**  
  - Type: Merge  
  - Role: Combines LinkedIn profile and post data into a single data structure.  
  - Inputs: Formatted LinkedIn data nodes.  
  - Outputs: Merged LinkedIn data for AI evaluation.

---

#### 2.3 Resume Processing

**Overview:**  
Handles the applicant’s resume PDF by converting it to text and preparing the text data for AI evaluation.

**Nodes Involved:**  
- Convert Pdf to Text(return text file link)  
- Get Resume Text data  

**Node Details:**  

- **Convert Pdf to Text(return text file link)**  
  - Type: PDFco API  
  - Role: Converts uploaded PDF resume to plain text, providing a text file link.  
  - Inputs: Resume PDF URL from form data.  
  - Outputs: Link to text file of resume content.  
  - Failures: PDF corruptions, unsupported formats, API limits.

- **Get Resume Text data**  
  - Type: HTTP Request  
  - Role: Fetches the converted resume text file content.  
  - Inputs: Text file link from PDFco node.  
  - Outputs: Raw resume text content.  
  - Failures: Broken links, network timeouts.

---

#### 2.4 AI Evaluation

**Overview:**  
Runs AI-powered evaluation on collected data (LinkedIn, GitHub, LeetCode, resume) using LangChain agents and OpenRouter language models. Each evaluation is parsed into structured outputs.

**Nodes Involved:**  
- OpenRouter Chat Model  
- OpenRouter Chat Model1  
- OpenRouter Chat Model2  
- OpenRouter Chat Model3  
- linkedin evaluation  
- github evaluation  
- leetcode evaluation  
- resume evaluation  
- Structured Output Parser  
- Structured Output Parser1  
- Structured Output Parser2  
- Structured Output Parser3  

**Node Details:**  

- **OpenRouter Chat Model / OpenRouter Chat Model1 / OpenRouter Chat Model2 / OpenRouter Chat Model3**  
  - Type: LangChain LM Chat OpenRouter  
  - Role: Language Model interface nodes configured to use OpenRouter API for AI processing.  
  - Inputs: Formatted data from respective domains (LinkedIn, GitHub, LeetCode, resume).  
  - Outputs: AI-generated textual evaluations.  
  - Failures: API auth errors, rate limits, response timeouts.

- **linkedin evaluation / github evaluation / leetcode evaluation / resume evaluation**  
  - Type: LangChain Agent  
  - Role: Custom AI agents applying evaluation logic on input data.  
  - Inputs: AI model output nodes.  
  - Outputs: Evaluation results.  
  - Failures: Agent logic errors or unexpected data format.

- **Structured Output Parser / Structured Output Parser1 / Structured Output Parser2 / Structured Output Parser3**  
  - Type: LangChain Structured Output Parser  
  - Role: Parses AI output into structured JSON or objects for consistent downstream processing.  
  - Inputs: AI agent outputs.  
  - Outputs: Structured evaluation data.  
  - Failures: Parsing failures due to unexpected AI output format.

---

#### 2.5 Aggregation and Output

**Overview:**  
This block merges all AI evaluation results, formats the combined data into a final structured output, and appends it to a Google Sheet.

**Nodes Involved:**  
- Merge All Evaluations  
- Format Final Output  
- Append Candidate Data to Google Sheet  

**Node Details:**  

- **Merge All Evaluations**  
  - Type: Merge  
  - Role: Combines outputs from LinkedIn, GitHub, LeetCode, and resume evaluations into a single object.  
  - Inputs: Outputs from all evaluation nodes.  
  - Outputs: Unified candidate evaluation data.  
  - Failures: Data structure mismatches.

- **Format Final Output**  
  - Type: Code (JavaScript)  
  - Role: Formats merged data into final structure suitable for Google Sheets insertion.  
  - Inputs: Merged evaluation data.  
  - Outputs: Final formatted candidate data.  
  - Failures: Formatting errors.

- **Append Candidate Data to Google Sheet**  
  - Type: Google Sheets  
  - Role: Appends a new row with candidate evaluation data to a specified Google Sheet.  
  - Inputs: Final formatted data.  
  - Outputs: Confirmation of row append.  
  - Failures: Google API auth errors, sheet permissions, rate limits.

---

### 3. Summary Table

| Node Name                         | Node Type                             | Functional Role                      | Input Node(s)                     | Output Node(s)                    | Sticky Note                     |
|----------------------------------|-------------------------------------|------------------------------------|----------------------------------|---------------------------------|--------------------------------|
| Trigger: New Job Application      | JotForm Trigger                     | Entry point for new applications   | —                                | Fetch JotForm Data              |                                |
| Fetch JotForm Data                | HTTP Request                       | Fetch detailed form submission data| Trigger: New Job Application     | Filter Required Fields          |                                |
| Filter Required Fields            | Code                              | Filter relevant application fields | Fetch JotForm Data               | Extract Usernames from Fields, Merge |                                |
| Extract Usernames from Fields     | Code                              | Extract usernames and resume link  | Filter Required Fields           | Validate GitHub Username, Validate Leetcode Username, Validate LinkedIn Username, Convert Pdf to Text(return text file link) |                                |
| Validate GitHub Username          | If (disabled)                     | Check if GitHub username present   | Extract Usernames from Fields    | Scrape GitHub Data, No Operation, do nothing1 |                                |
| Scrape GitHub Data                | HTTP Request                     | Scrape GitHub profile data         | Validate GitHub Username         | Format Github Data             |                                |
| Format Github Data                | Code                             | Format GitHub data for evaluation  | Scrape GitHub Data               | github evaluation             |                                |
| Validate Leetcode Username        | If (disabled)                     | Check if LeetCode username present | Extract Usernames from Fields    | Scrape LeetCode Stats, No Operation, do nothing2 |                                |
| Scrape LeetCode Stats             | HTTP Request                     | Scrape LeetCode user stats         | Validate Leetcode Username       | Format LeetCode Data           |                                |
| Format LeetCode Data              | Code                             | Format LeetCode stats for evaluation| Scrape LeetCode Stats           | leetcode evaluation           |                                |
| Validate LinkedIn Username        | If (disabled)                     | Check if LinkedIn username present | Extract Usernames from Fields    | Scrape LinkedIn Data, Scrape LinkedIn Post Data, No Operation, do nothing |                                |
| Scrape LinkedIn Data              | HTTP Request                     | Scrape LinkedIn profile data       | Validate LinkedIn Username       | Format LinkedIn Result        |                                |
| Scrape LinkedIn Post Data         | HTTP Request                     | Scrape LinkedIn posts data         | Validate LinkedIn Username       | Format LinkedIn Result1       |                                |
| Format LinkedIn Result            | Code                             | Format LinkedIn profile data       | Scrape LinkedIn Data             | Merge                        |                                |
| Format LinkedIn Result1           | Code                             | Format LinkedIn posts data          | Scrape LinkedIn Post Data        | Merge                        |                                |
| Merge                            | Merge                            | Merge LinkedIn profile and posts   | Format LinkedIn Result, Format LinkedIn Result1, Filter Required Fields | linkedin evaluation           |                                |
| Convert Pdf to Text(return text file link) | PDFco API                      | Convert resume PDF to text file link| Extract Usernames from Fields   | Get Resume Text data          |                                |
| Get Resume Text data             | HTTP Request                     | Fetch resume text content           | Convert Pdf to Text             | resume evaluation            |                                |
| OpenRouter Chat Model            | LangChain LM Chat OpenRouter      | AI language model for resume eval  | resume evaluation (ai_languageModel) | resume evaluation           |                                |
| OpenRouter Chat Model1           | LangChain LM Chat OpenRouter      | AI language model for GitHub eval  | github evaluation (ai_languageModel) | github evaluation           |                                |
| OpenRouter Chat Model2           | LangChain LM Chat OpenRouter      | AI language model for LeetCode eval| leetcode evaluation (ai_languageModel)| leetcode evaluation         |                                |
| OpenRouter Chat Model3           | LangChain LM Chat OpenRouter      | AI language model for LinkedIn eval| linkedin evaluation (ai_languageModel)| linkedin evaluation         |                                |
| resume evaluation                | LangChain Agent                   | AI agent for resume evaluation     | OpenRouter Chat Model (ai_languageModel), Structured Output Parser | Merge All Evaluations       |                                |
| github evaluation               | LangChain Agent                   | AI agent for GitHub evaluation     | OpenRouter Chat Model1 (ai_languageModel), Structured Output Parser2 | Merge All Evaluations       |                                |
| leetcode evaluation             | LangChain Agent                   | AI agent for LeetCode evaluation   | OpenRouter Chat Model2 (ai_languageModel), Structured Output Parser1 | Merge All Evaluations       |                                |
| linkedin evaluation             | LangChain Agent                   | AI agent for LinkedIn evaluation   | OpenRouter Chat Model3 (ai_languageModel), Structured Output Parser3 | Merge All Evaluations       |                                |
| Structured Output Parser        | LangChain Structured Output Parser| Parse AI resume evaluation output  | resume evaluation               | resume evaluation             |                                |
| Structured Output Parser1       | LangChain Structured Output Parser| Parse AI LeetCode evaluation output| leetcode evaluation             | leetcode evaluation           |                                |
| Structured Output Parser2       | LangChain Structured Output Parser| Parse AI GitHub evaluation output  | github evaluation              | github evaluation             |                                |
| Structured Output Parser3       | LangChain Structured Output Parser| Parse AI LinkedIn evaluation output| linkedin evaluation            | linkedin evaluation           |                                |
| Merge All Evaluations           | Merge                            | Merge all AI evaluation results    | linkedin evaluation, github evaluation, leetcode evaluation, resume evaluation | Format Final Output         |                                |
| Format Final Output             | Code                             | Format final merged data for sheets| Merge All Evaluations           | Append Candidate Data to Google Sheet |                                |
| Append Candidate Data to Google Sheet | Google Sheets                   | Append candidate data to sheet     | Format Final Output             | —                            |                                |
| No Operation, do nothing        | NoOp                             | Placeholder for bypassed validation| Validate LinkedIn Username      | —                            |                                |
| No Operation, do nothing1       | NoOp                             | Placeholder for bypassed validation| Validate GitHub Username        | —                            |                                |
| No Operation, do nothing2       | NoOp                             | Placeholder for bypassed validation| Validate Leetcode Username      | —                            |                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node**  
   - Add **JotForm Trigger** node, configure webhook ID or create new webhook to listen for new job applications.

2. **Fetch JotForm Submission Data**  
   - Add **HTTP Request** node to query JotForm API for detailed submission data using the submission ID from trigger output. Configure authentication for JotForm API.

3. **Filter Required Fields**  
   - Add **Code** node to process the fetched form data and filter only necessary fields such as LinkedIn, GitHub, LeetCode usernames, and resume link.

4. **Extract Usernames from Fields**  
   - Add **Code** node to parse and normalize usernames from URLs or text inputs. Extract resume PDF URL.

5. **Validate Usernames (optional, currently disabled)**  
   - Create **If** nodes for each username type (GitHub, LeetCode, LinkedIn) to check for existence or format. Connect true branch to scraping nodes; false branch to No Operation nodes.

6. **Scrape LinkedIn Data**  
   - Add two **HTTP Request** nodes: one for LinkedIn profile data, one for LinkedIn posts/activity.  
   - Add two **Code** nodes to format profile and post data.  
   - Add **Merge** node to combine formatted LinkedIn profile and post data.

7. **Scrape GitHub Data**  
   - Add **HTTP Request** node to scrape GitHub profile data.  
   - Add **Code** node to format GitHub data.

8. **Scrape LeetCode Stats**  
   - Add **HTTP Request** node to scrape LeetCode user stats.  
   - Add **Code** node to format LeetCode data.

9. **Resume Processing**  
   - Add **PDFco API** node to convert resume PDF to text file link.  
   - Add **HTTP Request** node to fetch raw text from the generated link.

10. **AI Model Setup**  
    - Add four **LangChain LM Chat OpenRouter** nodes, one for each data source (LinkedIn, GitHub, LeetCode, Resume). Configure with OpenRouter credentials and appropriate prompt settings.

11. **AI Agents**  
    - Add four **LangChain Agent** nodes to run evaluation logic on each data source, connected to respective OpenRouter model nodes.

12. **Structured Output Parsing**  
    - Add four **LangChain Structured Output Parser** nodes, each parsing the output from the AI agents to produce structured evaluation data.

13. **Merge Evaluations**  
    - Add **Merge** node to combine the four structured evaluation outputs into one.

14. **Format Final Output**  
    - Add **Code** node to format merged data into rows and columns suitable for Google Sheets.

15. **Google Sheets Append**  
    - Add **Google Sheets** node configured to append a new row to the target spreadsheet with the formatted candidate evaluation data. Configure OAuth2 credentials for Google Sheets.

16. **Connect all nodes sequentially** as per above flow, ensuring outputs feed into inputs correctly.

17. **Test the workflow** with sample application data for validation.

---

### 5. General Notes & Resources

| Note Content                                                                                       | Context or Link                                  |
|--------------------------------------------------------------------------------------------------|-------------------------------------------------|
| The workflow integrates JotForm for application intake and Google Sheets for data storage.       | Forms and spreadsheet integration                |
| Uses OpenRouter API as the language model backend via LangChain nodes for AI evaluation.          | AI evaluation component                          |
| PDFco API converts PDF resumes to text for AI processing.                                        | Resume preprocessing                            |
| Disabled validation nodes indicate username presence checks can be optionally enabled for robustness. | Optional input validation                         |
| Potential scraping challenges include captchas, login requirements, and API rate limits.         | Scraping LinkedIn, GitHub, LeetCode              |
| Documentation on LangChain and OpenRouter nodes can be found in official n8n community resources.| https://docs.n8n.io/                             |
| Ensure API credentials for JotForm, OpenRouter, PDFco, and Google Sheets are properly configured.| Credential management                             |

---

This structured reference document provides a detailed understanding, node-by-node analysis, and instructions for recreating the **TalentFlow AI** workflow to automate instant evaluation of applicant profiles and resumes using AI.