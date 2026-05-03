Generate Weekly Animal Rights Action Briefings from U.S. House Bills with Gemini AI

https://n8nworkflows.xyz/workflows/generate-weekly-animal-rights-action-briefings-from-u-s--house-bills-with-gemini-ai-6472


# Generate Weekly Animal Rights Action Briefings from U.S. House Bills with Gemini AI

### 1. Workflow Overview

This workflow automates the generation of weekly animal rights action briefings derived from U.S. House bills. It is designed for advocacy groups or activists focused on animal welfare, providing them with timely, AI-analyzed insights and prioritized action items based on the latest legislative bills. The workflow logically divides into the following blocks:

- **1.1 Weekly Scheduler**: Initiates the process every Monday morning.
- **1.2 Bill Fetching and PDF Extraction**: Downloads and parses the list of weekly House bills, extracting PDF links.
- **1.3 PDF Link Cleanup and Download**: Cleans the extracted links, splits them for individual processing, and downloads each PDF.
- **1.4 AI Bill Analysis (Gemini AI)**: Sends each bill PDF to Google Gemini AI for in-depth animal rights-focused analysis and returns structured JSON data.
- **1.5 Bill Filtering**: Filters analyzed bills based on relevance and urgency thresholds.
- **1.6 Research & Intelligence Gathering (Subworkflow)**: Enriches filtered bills with additional research on sponsors, lobbyists, advocacy groups, and media.
- **1.7 Email Briefing Generation**: Creates a structured HTML briefing email summarizing all relevant bills, prioritizing action points and including contact info.
- **1.8 Email Dispatch**: Sends the completed briefing email to the activist mailing list.

---

### 2. Block-by-Block Analysis

#### 2.1 Weekly Scheduler

- **Overview:** Triggers the entire workflow once a week, every Monday at 8:01 AM.
- **Nodes Involved:**  
  - Schedule Trigger  
  - Get Weekly Bills

- **Node Details:**  
  - **Schedule Trigger**  
    - *Type:* Schedule Trigger (built-in n8n node)  
    - *Configuration:* Runs weekly on Monday at 08:01  
    - *Input:* None  
    - *Output:* Triggers next node  
    - *Edge Cases:* Scheduler misfire if server down or n8n not running  
  - **Get Weekly Bills**  
    - *Type:* HTTP Request  
    - *Configuration:* Fetches the House floor webpage (https://docs.house.gov/floor/) containing bill info  
    - *Input:* Trigger from Schedule Trigger  
    - *Output:* HTML page data for bill extraction  
    - *Edge Cases:* HTTP errors, page format changes could break parsing

#### 2.2 Bill Fetching and PDF Extraction

- **Overview:** Extracts PDF links pointing to the full text of bills from the House floor HTML page.
- **Nodes Involved:**  
  - Extract PDF Links  
  - OpenRouter Chat Model (used for cleaning links)  
  - Clean PDF Links

- **Node Details:**  
  - **Extract PDF Links**  
    - *Type:* HTML Extractor  
    - *Configuration:* Extracts `.files` CSS selector values as an array of PDF URLs  
    - *Input:* HTML from "Get Weekly Bills"  
    - *Output:* Raw PDF link array  
    - *Edge Cases:* Changes in webpage structure can cause extraction failures  
  - **OpenRouter Chat Model**  
    - *Type:* Langchain OpenRouter Chat LLM node  
    - *Configuration:* Uses "google/gemini-2.5-flash" model to clean and normalize PDF URLs  
    - *Input:* Raw PDF links as JSON string  
    - *Output:* Cleaned, filtered PDF links  
    - *Credentials:* OpenRouter API  
    - *Edge Cases:* API rate limits, malformed input, response schema mismatch  
  - **Clean PDF Links**  
    - *Type:* Langchain Information Extractor  
    - *Configuration:* Extracts only valid PDF links; removes duplicates and irrelevant URLs  
    - *Input:* Output from OpenRouter Chat Model  
    - *Output:* Structured array of cleaned PDF URLs  
    - *Edge Cases:* Extraction failures if response format changes

#### 2.3 PDF Link Cleanup and Download

- **Overview:** Splits the cleaned array of PDF links into individual items and downloads each PDF file.
- **Nodes Involved:**  
  - Split Out Links  
  - Download PDF  
  - Convert PDF to Base64 String  
  - No Operation, do nothing (for error handling fallback)

- **Node Details:**  
  - **Split Out Links**  
    - *Type:* Split Out (array splitter)  
    - *Configuration:* Splits each PDF URL into a separate workflow item  
    - *Input:* Cleaned PDF link array  
    - *Output:* Individual PDF URLs per item  
  - **Download PDF**  
    - *Type:* HTTP Request  
    - *Configuration:* Downloads PDF binary from each URL  
    - *Input:* Single PDF URL item  
    - *Output:* PDF binary data  
    - *On Error:* Continue with "No Operation" to avoid halting workflow on download failure  
    - *Edge Cases:* Download failures, broken URLs, timeouts  
  - **Convert PDF to Base64 String**  
    - *Type:* Extract From File  
    - *Configuration:* Converts PDF binary to Base64 string in JSON property for AI consumption  
    - *Input:* PDF binary data  
    - *Output:* Base64 encoded PDF string

#### 2.4 AI Bill Analysis (Gemini AI)

- **Overview:** Sends each Base64-encoded PDF to Google Gemini AI for detailed animal rights-focused analysis.
- **Nodes Involved:**  
  - Analyze Bill with Gemini  
  - Set Analysis  
  - Check If Response Needed

- **Node Details:**  
  - **Analyze Bill with Gemini**  
    - *Type:* HTTP Request  
    - *Configuration:* POST to Gemini v2.0 API with a detailed prompt instructing AI to analyze bills strictly from an animal rights and anti-exploitation perspective.  
    - *Input:* Base64 PDF string  
    - *Output:* JSON object with fields such as bill_title, summary, animal_welfare_score, relevance_score, action_priority, stance, key_points_for_action  
    - *Authentication:* HTTP Query Authentication (API key)  
    - *Retry:* Enabled with 5s intervals for robustness  
    - *Edge Cases:* API errors, malformed PDFs, unexpected response schema  
  - **Set Analysis**  
    - *Type:* Set node  
    - *Configuration:* Extracts AI response text from Gemini result and assigns it to a JSON property `analysis`  
    - *Input:* Gemini API raw response  
    - *Output:* Structured analysis JSON  
  - **Check If Response Needed**  
    - *Type:* If node (conditional)  
    - *Configuration:* Checks if `relevance_score` > 0.5 AND `action_priority` > 0.5 to determine if bill merits further processing  
    - *Input:* Parsed analysis  
    - *Output:* If true, continues; else routes to noop  
    - *Edge Cases:* Missing or malformed scores could cause false negatives

#### 2.5 Bill Filtering and Aggregation

- **Overview:** Aggregates all bills passing the filter into a single collection for further research.
- **Nodes Involved:**  
  - Aggregate

- **Node Details:**  
  - **Aggregate**  
    - *Type:* Aggregate node  
    - *Configuration:* Aggregates all filtered bill items into one array in JSON data  
    - *Input:* Multiple individual bill analyses passing the filter  
    - *Output:* Combined array of bill data for batch processing

#### 2.6 Research & Intelligence Gathering (Subworkflow)

- **Overview:** Enhances the filtered bills with additional contextual research such as sponsor info, lobbying groups, advocacy organizations, and media sentiment.
- **Nodes Involved:**  
  - Research Bills (Subworkflow invocation)

- **Node Details:**  
  - **Research Bills**  
    - *Type:* Execute Workflow (Subworkflow)  
    - *Configuration:* Calls a separate research agent workflow designed for animal advocacy intelligence gathering  
    - *Input:* Aggregated array of bills  
    - *Output:* Enriched bill data with research insights  
    - *Notes:* Requires pre-importing and configuring the referenced subworkflow  
    - *Sticky Note:* Links to subworkflow: [Multi-Tool Research Agent for Animal Advocacy](https://n8n.io/workflows/5588-multi-tool-research-agent-for-animal-advocacy-with-openrouter-serper-and-open-paws-db/)  
    - *Edge Cases:* Subworkflow failure or misconfiguration could block enrichment

#### 2.7 Email Briefing Generation

- **Overview:** Generates a high-impact, HTML-formatted email briefing for activists using the enriched bill data.
- **Nodes Involved:**  
  - Write Email Alert  
  - OpenRouter Chat Model1  
  - Structured Output Parser

- **Node Details:**  
  - **Write Email Alert**  
    - *Type:* Langchain Chain LLM  
    - *Configuration:* Uses a detailed prompt to create an actionable, well-structured HTML email including subject, prioritized action dashboard, bill summaries, contacts, research highlights, and calls to action  
    - *Input:* Research-enriched bills from subworkflow and aggregated data  
    - *Output:* JSON with HTML email body and subject line  
    - *Edge Cases:* AI hallucination risk mitigated by prompt instructions to not invent data  
  - **OpenRouter Chat Model1**  
    - *Type:* Langchain OpenRouter Chat LLM  
    - *Configuration:* Uses Anthropic Claude Sonnet 4 model for email writing tasks  
    - *Input:* Prompt text and bill data  
    - *Output:* Raw AI response for email content  
    - *Credentials:* OpenRouter API  
  - **Structured Output Parser**  
    - *Type:* Langchain Structured Output Parser  
    - *Configuration:* Parses AI response into structured JSON with `subject_line` and `email_body_html`  
    - *Input:* Raw AI email output  
    - *Output:* Final parsed email content

#### 2.8 Email Dispatch

- **Overview:** Sends the finalized HTML email briefing to a predefined mailing list.
- **Nodes Involved:**  
  - Send email

- **Node Details:**  
  - **Send email**  
    - *Type:* Email Send (SMTP)  
    - *Configuration:*  
      - Uses SMTP credentials for ProtonMail account  
      - Sends to configured recipient email (placeholder: email@example.com)  
      - Subject and HTML body sourced from parsed AI output  
      - Attribution disabled for clean email  
    - *Input:* Parsed email content  
    - *Output:* Email delivery status  
    - *Edge Cases:* SMTP authentication failures, invalid recipient addresses

---

### 3. Summary Table

| Node Name               | Node Type                           | Functional Role                                | Input Node(s)                | Output Node(s)             | Sticky Note                                                                                                         |
|-------------------------|-----------------------------------|------------------------------------------------|-----------------------------|----------------------------|---------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger        | Schedule Trigger                  | Weekly trigger to start workflow                | None                        | Get Weekly Bills            | **Weekly Scheduler**  Runs every Monday at 8 AM to start fetching bills.                                            |
| Get Weekly Bills        | HTTP Request                     | Downloads House floor page with bills           | Schedule Trigger            | Extract PDF Links           | **Bill Fetching & PDF Parsing**  Downloads bill list for the week and extracts PDFs.                                |
| Extract PDF Links       | HTML Extractor                   | Extracts raw PDF links from HTML page           | Get Weekly Bills            | Clean PDF Links             |                                                                                                                     |
| OpenRouter Chat Model   | Langchain OpenRouter Chat LLM   | Cleans and normalizes PDF links                  | Extract PDF Links           | Clean PDF Links             | **Bill Link Cleanup & PDF Download**  Cleans and normalizes links, removes duplicates.                              |
| Clean PDF Links         | Langchain Information Extractor | Extracts cleaned PDF links as array              | OpenRouter Chat Model       | Split Out Links             |                                                                                                                     |
| Split Out Links         | Split Out                       | Splits array of PDF links into individual items | Clean PDF Links             | Download PDF                |                                                                                                                     |
| Download PDF            | HTTP Request                    | Downloads individual PDF files                    | Split Out Links             | Convert PDF to Base64 String / No Operation | On error, continues without halting workflow.                                                                       |
| Convert PDF to Base64 String | Extract From File             | Converts PDF binary to Base64 string             | Download PDF                | Analyze Bill with Gemini    |                                                                                                                     |
| Analyze Bill with Gemini| HTTP Request                    | Sends PDF to Gemini AI for animal rights analysis| Convert PDF to Base64 String| Set Analysis / No Operation | **AI Bill Analysis (Gemini)**  Reads bills with animal rights perspective and returns structured JSON.              |
| Set Analysis            | Set                            | Extracts AI analysis JSON from Gemini response   | Analyze Bill with Gemini    | Check If Response Needed    |                                                                                                                     |
| Check If Response Needed| If                             | Filters bills by relevance and action priority   | Set Analysis                | Aggregate / No Operation    | **Bill Filtering**  Filters bills based on relevance and urgency thresholds.                                       |
| Aggregate               | Aggregate                      | Aggregates filtered bills into one batch          | Check If Response Needed    | Research Bills              |                                                                                                                     |
| Research Bills          | Execute Workflow (Subworkflow) | Enriches bill data with research intelligence     | Aggregate                   | Write Email Alert           | **Research & Intel Gathering**  Enriches bills with sponsor, lobbying, advocacy info. Requires subworkflow setup.  |
| Write Email Alert       | Langchain Chain LLM            | Generates HTML briefing email from enriched data | Research Bills / Structured Output Parser | Send email                 | **Email Writer**  Generates detailed, actionable email for activists.                                              |
| OpenRouter Chat Model1  | Langchain OpenRouter Chat LLM | Supports email generation using Anthropic Claude | Write Email Alert           | Structured Output Parser    |                                                                                                                     |
| Structured Output Parser| Langchain Output Parser        | Parses AI email response to structured JSON       | OpenRouter Chat Model1      | Write Email Alert           |                                                                                                                     |
| Send email              | Email Send (SMTP)              | Sends the final email briefing                     | Write Email Alert           | None                       | **Email Dispatch**  Sends briefing to mailing list.                                                                |
| No Operation, do nothing| No Operation                  | Error fallback or continue path                    | Download PDF / Analyze Bill with Gemini / Check If Response Needed | None                       |                                                                                                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Configure to run every week on Monday at 08:01.

2. **Create HTTP Request Node ("Get Weekly Bills")**  
   - Set URL to `https://docs.house.gov/floor/`  
   - Method: GET  
   - Connect from Schedule Trigger.

3. **Create HTML Extract Node ("Extract PDF Links")**  
   - Operation: Extract HTML content  
   - Extraction CSS selector: `.files`  
   - Return array: true  
   - Input: Output from "Get Weekly Bills".

4. **Add OpenRouter Chat Model Node for Link Cleaning**  
   - Model: `google/gemini-2.5-flash`  
   - Credentials: OpenRouter API account  
   - Input: JSON stringified links from Extract PDF Links  
   - Output: Pass to next node.

5. **Add Langchain Information Extractor Node ("Clean PDF Links")**  
   - Input schema expects an object with `PDF_Link` array of URIs  
   - Input: OpenRouter Chat Model output  
   - Output: Cleaned array of PDF URLs.

6. **Add Split Out Node ("Split Out Links")**  
   - Field to split: `output.PDF_Link`  
   - Input: Clean PDF Links output.

7. **Add HTTP Request Node ("Download PDF")**  
   - Method: GET  
   - URL: Use expression to pull current item PDF link  
   - On error: Continue (do not fail workflow)  
   - Input: Split Out Links output.

8. **Add Extract From File Node ("Convert PDF to Base64 String")**  
   - Operation: binaryToProperty  
   - Input: Download PDF output.

9. **Add HTTP Request Node ("Analyze Bill with Gemini")**  
   - URL: `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.0-flash:generateContent`  
   - Method: POST  
   - Authentication: HTTP Query Auth with API key  
   - Body: JSON with prompt instructing AI to analyze the bill PDF from animal rights perspective, passing Base64 PDF data inline  
   - Retry enabled with 5s wait  
   - Input: Base64 PDF from previous node.

10. **Add Set Node ("Set Analysis")**  
    - Extract AI JSON analysis text from Gemini response into `analysis` property.

11. **Add If Node ("Check If Response Needed")**  
    - Condition: `analysis.relevance_score` > 0.5 AND `analysis.action_priority` > 0.5  
    - True: continue flow  
    - False: connect to No Operation node.

12. **Add Aggregate Node ("Aggregate")**  
    - Aggregate all filtered bill data into an array.

13. **Add Execute Workflow Node ("Research Bills")**  
    - Select subworkflow: [Multi-Tool Research Agent for Animal Advocacy](https://n8n.io/workflows/5588-multi-tool-research-agent-for-animal-advocacy-with-openrouter-serper-and-open-paws-db/)  
    - Pass aggregated bill array as input.

14. **Add Langchain Chain LLM Node ("Write Email Alert")**  
    - Prompt: Detailed instructions to produce HTML email briefing with subject line, action dashboard, detailed bill sections, contacts, and calls to action.  
    - Input: Research Bills output and aggregated data.

15. **Add OpenRouter Chat Model Node ("OpenRouter Chat Model1")**  
    - Model: `anthropic/claude-sonnet-4`  
    - Credentials: OpenRouter API  
    - Input: Same as Write Email Alert for email generation.

16. **Add Structured Output Parser Node**  
    - Auto-fix enabled  
    - Example JSON schema: `{ "subject_line": "string", "email_body_html": "string" }`  
    - Input: OpenRouter Chat Model1 output.

17. **Connect Structured Output Parser output back to Write Email Alert for final processing.**

18. **Add Email Send Node ("Send email")**  
    - SMTP credentials (e.g., ProtonMail) configured  
    - Recipient: Your activist mailing list email address  
    - From: Your email address  
    - Subject: From parsed output `subject_line`  
    - HTML body: From parsed output `email_body_html`  
    - Input: Write Email Alert output.

19. **Add No Operation Nodes**  
    - For error handling after Download PDF, Analyze Bill with Gemini, and Check If Response Needed nodes to gracefully handle errors without stopping workflow.

20. **Connect the nodes according to the logical flow described, ensuring all data transformations and conditions are linked properly.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                | Context or Link                                                                                                         |
|-----------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------|
| **Weekly Scheduler:** Runs every Monday at 8 AM to start the bill fetching process.                                           | Sticky Note on Schedule Trigger node                                                                                    |
| Use the [Multi-Tool Research Agent for Animal Advocacy subworkflow](https://n8n.io/workflows/5588-multi-tool-research-agent-for-animal-advocacy-with-openrouter-serper-and-open-paws-db/) for research enrichment. | Research Bills node sticky note                                                                                         |
| Adjust AI prompt texts in the Gemini analysis node to tailor focus on different animal groups or priorities.                 | Sticky Note on AI Bill Analysis (Gemini)                                                                                |
| Filtering thresholds for relevance and action priority can be tuned in the If node to control how many bills are included.   | Sticky Note on Bill Filtering node                                                                                      |
| Email formatting and tone can be modified by editing the prompt in the Write Email Alert node.                               | Sticky Note on Email Writer node                                                                                        |
| Update recipient or SMTP settings in the Email Send node to match your mail provider and subscriber list.                    | Sticky Note on Email Dispatch node                                                                                      |
| The entire workflow is designed to be modular; you can replace the bill source URL or research subworkflow as needed.        | General advice from Bill Fetching & PDF Parsing sticky note                                                            |

---

**Disclaimer**  
The text provided is exclusively from an automated workflow created with n8n, respecting all content policies. It contains no illegal, offensive, or protected elements. All data processed is legal and publicly available.