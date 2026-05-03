Create AI-Powered SEO Audit Reports with DataForSEO and Claude

https://n8nworkflows.xyz/workflows/create-ai-powered-seo-audit-reports-with-dataforseo-and-claude-9502


# Create AI-Powered SEO Audit Reports with DataForSEO and Claude

### 1. Workflow Overview

The workflow "Generate AI-powered SEO audit reports from company websites" automates the end-to-end process of producing detailed SEO audit reports for client websites using DataForSEO API for data extraction and Claude (via OpenRouter) for AI-powered analysis and report generation. It serves digital marketing agencies, SEO consultants, and SaaS teams who require fast, automated, and professional SEO audits.

The logical structure is divided into the following blocks:

- **1.1 Input Reception and Initialization:** Collects user input via webhook and sets initial parameters.
- **1.2 Website Content Scraping & Data Collection:** Uses DataForSEO APIs to extract website content and perform technical SEO analysis.
- **1.3 AI Research & Strategic Analysis:** Leverages large language models (LLMs) to analyze business, market position, and product strategy.
- **1.4 SEO Data Aggregation and Waiting:** Aggregates SEO data and introduces wait time to ensure complete data availability.
- **1.5 SEO Audit Report Generation:** AI agent generates a detailed, structured HTML SEO audit report based on aggregated data and AI research.
- **1.6 PDF Report Creation:** Converts the HTML report to PDF format, adding branded cover and closing pages.
- **1.7 Report Delivery:** Sends the generated PDF report via email to the specified recipient.
- **1.8 Error Handling and URL Validation:** Handles edge cases such as invalid URLs or websites that cannot be scraped.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Initialization

**Overview:**  
Receives HTTP POST requests containing user data (company info) and initializes variables for downstream processing.

**Nodes Involved:**  
- Webhook1  
- SET_USER_INPUT  
- Execute Workflow

**Node Details:**

- **Webhook1:**  
  - Type: Webhook  
  - Role: Entry point receiving user data (full name, email, company name, website URL, phone) via POST.  
  - Config: HTTP POST, path "27ea5610-f693-4370-a0c9-8ceb5253a49d"  
  - Outputs: User payload forwarded to SET_USER_INPUT.  
  - Edge Cases: Invalid or incomplete POST body may cause missing data; ensure client sends all required fields.

- **SET_USER_INPUT:**  
  - Type: Set  
  - Role: Maps webhook JSON body fields to named variables for clarity and downstream use (Full_Name, Email, Phone_Number, Compnay_Name, Website_URL).  
  - Config: Assign fields directly from webhook body.  
  - Inputs: Webhook1  
  - Outputs: Structured input data for workflow execution node.  
  - Edge Cases: Typo in "Compnay_Name" variable (likely intentional for backward compatibility).

- **Execute Workflow:**  
  - Type: Execute Workflow (sub-workflow)  
  - Role: Calls the SEO Website Audit System sub-workflow, passing the Website_URL as parameter `query`.  
  - Config: Wait for sub-workflow completion before continuing.  
  - Inputs: SET_USER_INPUT  
  - Outputs: Workflow result forwarded to Set_Prompt node.  
  - Edge Cases: Sub-workflow failure or timeout; ensure sub-workflow is published and accessible.

---

#### 2.2 Website Content Scraping & Data Collection

**Overview:**  
Scrapes the target website content and gathers SEO metrics using DataForSEO APIs. Converts raw HTML to markdown for AI processing.

**Nodes Involved:**  
- Crawler Request  
- Wait5  
- Website SEO Data1  
- Aggregate  
- DATA FOR SEO  
- DATA FOR SEO1  
- Markdown1  
- Edit Fields29  
- Edit Fields30  
- Edit Fields31  
- Edit Fields34  
- If https?

**Node Details:**

- **Crawler Request:**  
  - Type: HTTP Request  
  - Role: Initiates DataForSEO on_page task to crawl the target website, limited to 4 pages max and depth 5.  
  - Config: POST JSON body includes target URL from user input.  
  - Credentials: DataForSEO (HTTP Basic Auth)  
  - Outputs: Task ID for crawling, triggers Wait5.  
  - Edge Cases: API rate limits, site blocking crawler, invalid URL errors.

- **Wait5:**  
  - Type: Wait  
  - Role: Pauses workflow for 3 minutes to allow crawl completion and data availability.  
  - Inputs: Crawler Request  
  - Outputs: Website SEO Data1  
  - Edge Cases: Fixed wait time may be insufficient for large sites; consider adaptive polling.

- **Website SEO Data1:**  
  - Type: HTTP Request  
  - Role: Requests on_page pages data for the crawl task using the task ID from Crawler Request results.  
  - Config: POST JSON body with task ID, authenticated with DataForSEO credentials.  
  - Outputs: SEO data for pages.  
  - Edge Cases: Task ID expiration, incomplete crawl data.

- **Aggregate:**  
  - Type: Aggregate  
  - Role: Collects and combines all SEO data items into a single array under "output_data".  
  - Inputs: Website SEO Data1  
  - Outputs: Aggregated SEO data for AI consumption.

- **DATA FOR SEO:**  
  - Type: HTTP Request  
  - Role: Requests instant page analysis from DataForSEO for the specified URL (site).  
  - Config: POST with options enabling JavaScript, browser rendering, raw HTML storage.  
  - Credentials: DataForSEO  
  - Outputs: Task ID for raw HTML retrieval.  
  - Inputs: Edit Fields29 or Edit Fields30 (site URL adjusted).

- **DATA FOR SEO1:**  
  - Type: HTTP Request  
  - Role: Retrieves raw HTML for the instant page task using task ID and URL.  
  - Config: POST with task ID and URL.  
  - Outputs: Raw HTML data.  
  - Edge Cases: Task ID invalid or expired, site restrictions.

- **Markdown1:**  
  - Type: Markdown  
  - Role: Converts HTML content from DataForSEO1 to markdown format, ignoring images and links for cleaner text.  
  - Inputs: DATA FOR SEO1  
  - Outputs: Markdown content for AI research.

- **Edit Fields29 / Edit Fields30:**  
  - Type: Set  
  - Role: Normalize the site URL by checking if it contains "http" or not; adds https prefix if missing.  
  - Inputs: If https?  
  - Outputs: Site URL for DATA FOR SEO node.

- **Edit Fields31:**  
  - Type: Set  
  - Role: Sets "Response" field to raw HTML data from DATA FOR SEO1.  
  - Inputs: Markdown1  
  - Outputs: For business research prompt.

- **Edit Fields34:**  
  - Type: Set  
  - Role: Sets error message if the website cannot be scraped, differentiating between research agent and business overview contexts.  
  - Inputs: Markdown1 (on error branch), Markdown1 (on continue)  
  - Outputs: Error message used in AI prompt.

- **If https?:**  
  - Type: If  
  - Role: Branches workflow based on whether the URL includes "http" prefix.  
  - Inputs: When Executed by Another Workflow trigger  
  - Outputs: Edit Fields29 (true) or Edit Fields30 (false).

---

#### 2.3 AI Research & Strategic Analysis

**Overview:**  
Processes scraped data with AI models to generate detailed business research and product strategy content forming the basis for the SEO audit report.

**Nodes Involved:**  
- Set_Prompt  
- Business Research  
- Product Strategy  
- OpenRouter Chat Model1

**Node Details:**

- **Set_Prompt:**  
  - Type: Set  
  - Role: Constructs a detailed prompt for the Business Research AI agent, embedding scraped content and company info.  
  - Inputs: Execute Workflow output (scraped data)  
  - Outputs: User prompt string for LLM.

- **Business Research:**  
  - Type: OpenAI (GPT-4o-search-preview)  
  - Role: Performs deep investigative research on the company, synthesizing online data beyond the website to create a comprehensive business overview.  
  - Config: System role set to Internet researcher with focus on accurate marketing data.  
  - Inputs: Set_Prompt  
  - Outputs: Business overview content for Product Strategy node.

- **Product Strategy:**  
  - Type: ChainLLM  
  - Role: Generates a strategic content architecture plan with hierarchical content strategy (BOFU → MOFU → TOFU), optimized for AI and traditional SEO.  
  - Config: Prompt includes business overview from Business Research and detailed instructions for content plan creation.  
  - Inputs: Business Research output  
  - Outputs: Product strategy text for crawling.

- **OpenRouter Chat Model1:**  
  - Type: OpenRouter LLM (Gemini 2.5 Flash)  
  - Role: AI language model used for Product Strategy node execution.  
  - Credentials: OpenRouter API with specific project creds.  
  - Edge Cases: API limits, network errors.

---

#### 2.4 SEO Data Aggregation and Waiting

**Overview:**  
Ensures complete SEO data collection by waiting and aggregating page data for final processing.

**Nodes Involved:**  
- Wait5  
- Website SEO Data1  
- Aggregate

**See 2.2 for detailed node descriptions.**

---

#### 2.5 SEO Audit Report Generation

**Overview:**  
Uses an AI chain to generate a highly detailed, structured HTML SEO audit report based on combined business research, product strategy, and aggregated SEO data.

**Nodes Involved:**  
- Sample_Code  
- Report_Generator_Agent  
- Structured Output Parser1  
- PDF_First_Page  
- PDF_Last_Page1

**Node Details:**

- **Sample_Code:**  
  - Type: Set  
  - Role: Provides a comprehensive HTML + CSS template example for the audit report's internal pages, including table of contents and multiple sections.  
  - Outputs: HTML template string for Report Generator prompt.

- **Report_Generator_Agent:**  
  - Type: ChainLLM (Claude Opus 4.1 via OpenRouter)  
  - Role: Generates the entire website audit report HTML content inside a JSON object, following strict formatting and section guidelines.  
  - Config: Receives input data fields including business overview, product strategy, aggregated SEO data, first/last page templates, and sample code for reference.  
  - Outputs: JSON object with "report_type", "format", and "content" fields containing the full audit report HTML.  
  - Edge Cases: LLM prompt failures, partial outputs, or format errors.

- **Structured Output Parser1:**  
  - Type: Structured Output Parser (Langchain)  
  - Role: Validates and parses the AI-generated JSON output according to a strict JSON schema ensuring report correctness.  
  - Inputs: Report_Generator_Agent output  
  - Outputs: Parsed JSON report content.

- **PDF_First_Page / PDF_Last_Page1:**  
  - Type: Set  
  - Role: Define branded first and last page HTML templates used in the final PDF report.  
  - Outputs: HTML strings injected into report generation prompt.

---

#### 2.6 PDF Report Creation

**Overview:**  
Converts the generated HTML SEO audit report into a PDF document using PDF.co API.

**Nodes Involved:**  
- PDFco Api  
- Get_Pdf_File

**Node Details:**

- **PDFco Api:**  
  - Type: PDFco API Node  
  - Role: Converts HTML report content to PDF format.  
  - Config: Receives HTML from Report_Generator_Agent output, names the PDF after the company.  
  - Credentials: PDF.co API account.  
  - Retry enabled with 3 seconds delay between tries.  
  - Edge Cases: API rate limits, HTML conversion errors.

- **Get_Pdf_File:**  
  - Type: HTTP Request  
  - Role: Downloads the generated PDF file from PDF.co service using the returned URL.  
  - Inputs: PDFco Api output (URL field).  
  - Outputs: Binary PDF data forwarded to email node.  
  - Edge Cases: File unavailable or expired URL.

---

#### 2.7 Report Delivery

**Overview:**  
Sends the generated PDF audit report as an email attachment to the specified recipient.

**Nodes Involved:**  
- Send a message

**Node Details:**

- **Send a message:**  
  - Type: Gmail Node  
  - Role: Sends an email with the PDF audit report attached to the user-provided email address.  
  - Config:  
    - Recipient: Email from SET_USER_INPUT  
    - Subject: Fixed "Your Website Audit"  
    - Message: HTML formatted email body with branding and call to action.  
    - Attachments: PDF binary data from Get_Pdf_File.  
    - Sender Name: "1prompt License"  
  - Credentials: Gmail OAuth2 credentials configured.  
  - Edge Cases: Gmail quota limits, invalid email addresses, attachment size limits.

---

#### 2.8 Error Handling and URL Validation

**Overview:**  
Handles situations where the input URL is missing protocol or the website cannot be scraped. Provides user-friendly fallback messages.

**Nodes Involved:**  
- If https?  
- Edit Fields29  
- Edit Fields30  
- Edit Fields34  
- Markdown1 (error continuation)

**Node Details:**

- **If https?:**  
  - Checks if URL contains "http"  
  - Branches to prepend https or pass through URL as is.

- **Edit Fields29 / Edit Fields30:**  
  - Normalize URL accordingly.

- **Edit Fields34:**  
  - Sets error message when scraping fails, differentiates messages based on workflow context.

- **Markdown1:**  
  - Continues on error to prevent workflow crash and allow error message propagation.

---

### 3. Summary Table

| Node Name                 | Node Type                         | Functional Role                                   | Input Node(s)                | Output Node(s)                 | Sticky Note                                                                                              |
|---------------------------|----------------------------------|--------------------------------------------------|------------------------------|-------------------------------|--------------------------------------------------------------------------------------------------------|
| Webhook1                  | Webhook                         | Receive user input via HTTP POST                  |                              | SET_USER_INPUT                |                                                                                                        |
| SET_USER_INPUT             | Set                             | Map input data fields                             | Webhook1                     | Execute Workflow              |                                                                                                        |
| Execute Workflow           | Execute Workflow                | Call SEO Website Audit System sub-workflow       | SET_USER_INPUT               | Set_Prompt                   |                                                                                                        |
| Set_Prompt                | Set                             | Prepare AI prompt for business research          | Execute Workflow             | Business Research            |                                                                                                        |
| Business Research          | OpenAI (GPT-4o-search-preview)  | Generate detailed business research insights     | Set_Prompt                  | Product Strategy             |                                                                                                        |
| Product Strategy           | ChainLLM + OpenRouter LLM       | Generate strategic content architecture           | Business Research            | Crawler Request              |                                                                                                        |
| Crawler Request            | HTTP Request                   | Initiate DataForSEO crawl on website             | Product Strategy             | Wait5                        |                                                                                                        |
| Wait5                     | Wait                            | Wait for crawl completion                         | Crawler Request              | Website SEO Data1            |                                                                                                        |
| Website SEO Data1          | HTTP Request                   | Retrieve SEO page data                            | Wait5                       | Aggregate                   |                                                                                                        |
| Aggregate                 | Aggregate                      | Aggregate SEO data                               | Website SEO Data1            | PDF_First_Page               |                                                                                                        |
| PDF_First_Page             | Set                             | Provide branded first page HTML                   | Aggregate                   | PDF_Last_Page1               |                                                                                                        |
| PDF_Last_Page1             | Set                             | Provide branded last page HTML                    | PDF_First_Page              | Sample_Code                  |                                                                                                        |
| Sample_Code                | Set                             | Provide HTML template example for audit report   | PDF_Last_Page1              | Report_Generator_Agent       |                                                                                                        |
| Report_Generator_Agent     | ChainLLM (Claude Opus 4.1)      | Generate full detailed SEO audit report in HTML  | Sample_Code, Business Research, Product Strategy, Aggregate | PDFco Api                   | Sticky Note11: AI Analysis & Strategic Insights; Sticky Note15: Collect SEO Metrics                   |
| Structured Output Parser1  | Structured Output Parser        | Validate and parse AI JSON report output          | Report_Generator_Agent      | PDFco Api                   |                                                                                                        |
| PDFco Api                 | PDFco API Node                 | Convert HTML report to PDF                        | Report_Generator_Agent      | Get_Pdf_File                |                                                                                                        |
| Get_Pdf_File               | HTTP Request                   | Download generated PDF file                       | PDFco Api                  | Send a message              |                                                                                                        |
| Send a message             | Gmail Node                    | Email the PDF report to client                    | Get_Pdf_File               |                               |                                                                                                        |
| If https?                 | If                             | Check if URL contains http protocol               | When Executed by Another Workflow | Edit Fields29, Edit Fields30 |                                                                                                        |
| Edit Fields29              | Set                             | Use URL as is if contains http                    | If https? (true)            | DATA FOR SEO                |                                                                                                        |
| Edit Fields30              | Set                             | Prepend https to URL if missing                    | If https? (false)           | DATA FOR SEO                |                                                                                                        |
| DATA FOR SEO              | HTTP Request                   | Request instant page SEO data                      | Edit Fields29/Edit Fields30 | DATA FOR SEO1               |                                                                                                        |
| DATA FOR SEO1             | HTTP Request                   | Retrieve raw HTML for instant page task            | DATA FOR SEO                | Markdown1, Edit Fields34     |                                                                                                        |
| Markdown1                 | Markdown                       | Convert HTML to markdown                           | DATA FOR SEO1               | Edit Fields31, Edit Fields34 |                                                                                                        |
| Edit Fields31              | Set                             | Store raw HTML response                            | Markdown1                  | Set_Prompt                  |                                                                                                        |
| Edit Fields34              | Set                             | Error message if site cannot be scraped            | DATA FOR SEO1, Markdown1 (error) | Set_Prompt (error path)     |                                                                                                        |
| When Executed by Another Workflow | Execute Workflow Trigger | Entry trigger for sub-workflow                     |                              | If https?                   | Sticky Note: Scraper Sub-Workflow                                                                      |
| Sticky Note (Multiple)     | Sticky Note                    | Provides documentation and setup notes             |                              |                               | Sticky Note12: Automated SEO Intelligence Platform explanation; Sticky Note13: API Rate Limits warning |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node ("Webhook1"):**  
   - Type: webhook  
   - HTTP Method: POST  
   - Path: unique identifier (e.g., "27ea5610-f693-4370-a0c9-8ceb5253a49d")  
   - Purpose: Receive JSON POST with user info (fullName, email, phoneNumber, companyName, companyWebsite).

2. **Add Set Node ("SET_USER_INPUT"):**  
   - Map JSON body fields to variables:  
     - Full_Name = body.fullName  
     - Email = body.email  
     - Phone_Number = body.phoneNumber  
     - Compnay_Name = body.companyName (note the spelling)  
     - Website_URL = body.companyWebsite  
   - Connect Webhook1 → SET_USER_INPUT

3. **Add Execute Workflow Node ("Execute Workflow"):**  
   - Configure to call sub-workflow "SEO Website Audit System" (provide workflow ID)  
   - Pass parameter `query` = `={{ $json.Website_URL }}`  
   - Enable "Wait for sub-workflow" option  
   - Connect SET_USER_INPUT → Execute Workflow

4. **Inside SEO Website Audit System Sub-Workflow (not detailed here):**  
   - Implement scraping, SEO data collection, AI analysis, report generation, PDF creation, and email sending as per blocks below.

5. **In Main Workflow, after Execute Workflow, create Set Node ("Set_Prompt"):**  
   - Compose AI prompt for business research using scraped raw HTML (`Response` field) and company info.  
   - Connect Execute Workflow → Set_Prompt

6. **Add OpenAI Node ("Business Research"):**  
   - Model: GPT-4o-search-preview  
   - System prompt: Researcher role focused on detailed marketing intel  
   - Pass prompt from Set_Prompt  
   - Connect Set_Prompt → Business Research

7. **Add ChainLLM Node ("Product Strategy"):**  
   - Use OpenRouter LLM (Gemini 2.5 Flash) with configured credentials  
   - Generate strategic content architecture based on Business Research output  
   - Connect Business Research → Product Strategy

8. **Add HTTP Request Node ("Crawler Request"):**  
   - URL: DataForSEO on_page task_post API  
   - POST JSON body with target URL from `SET_USER_INPUT.Website_URL`, max pages=4, max depth=5  
   - Auth: DataForSEO HTTP Basic  
   - Connect Product Strategy → Crawler Request

9. **Add Wait Node ("Wait5"):**  
   - Wait for 3 minutes to allow crawl completion  
   - Connect Crawler Request → Wait5

10. **Add HTTP Request Node ("Website SEO Data1"):**  
    - URL: DataForSEO on_page/pages API  
    - POST JSON body with task ID from Crawler Request result  
    - Auth: DataForSEO HTTP Basic  
    - Connect Wait5 → Website SEO Data1

11. **Add Aggregate Node ("Aggregate"):**  
    - Aggregate all SEO data items into single field `output_data`  
    - Connect Website SEO Data1 → Aggregate

12. **Add Set Nodes ("PDF_First_Page", "PDF_Last_Page1"):**  
    - Set branded HTML templates for first and last pages of PDF report  
    - Connect Aggregate → PDF_First_Page → PDF_Last_Page1

13. **Add Set Node ("Sample_Code"):**  
    - Store comprehensive HTML/CSS template example for audit report body  
    - Connect PDF_Last_Page1 → Sample_Code

14. **Add ChainLLM Node ("Report_Generator_Agent"):**  
    - Use Claude Opus 4.1 via OpenRouter  
    - Input: Business Research, Product Strategy, aggregated SEO data, first/last page templates, sample code  
    - Output: JSON object with full audit report HTML  
    - Connect Sample_Code & Business Research & Product Strategy & Aggregate → Report_Generator_Agent

15. **Add Structured Output Parser Node ("Structured Output Parser1"):**  
    - Validate JSON report output structure against defined schema  
    - Connect Report_Generator_Agent → Structured Output Parser1

16. **Add PDFco API Node ("PDFco Api"):**  
    - Convert HTML report content (from parsed JSON) to PDF  
    - Set PDF file name from company name variable  
    - Connect Structured Output Parser1 → PDFco Api

17. **Add HTTP Request Node ("Get_Pdf_File"):**  
    - Download generated PDF file from PDFco URL  
    - Connect PDFco Api → Get_Pdf_File

18. **Add Gmail Node ("Send a message"):**  
    - Configure to send email to user Email from SET_USER_INPUT  
    - Subject: "Your Website Audit"  
    - HTML body: branded email template referencing user name and company  
    - Attach PDF file from Get_Pdf_File binary data  
    - Configure Gmail OAuth2 credentials  
    - Connect Get_Pdf_File → Send a message

19. **Add URL Validation Nodes (If https?, Edit Fields29, Edit Fields30):**  
    - Check if Website_URL starts with "http"  
    - If not, prepend "https://"  
    - Connect When Executed by Another Workflow trigger (for sub-workflow entry) → If https? → Edit Fields29 or Edit Fields30 → DATA FOR SEO

20. **Add DataForSEO Instant Pages and Raw HTML nodes ("DATA FOR SEO", "DATA FOR SEO1"):**  
    - Request instant page SEO data, enable browser rendering and raw HTML storage  
    - Retrieve raw HTML content for scraping  
    - Connect Edit Fields29/Edit Fields30 → DATA FOR SEO → DATA FOR SEO1

21. **Add Markdown Node ("Markdown1"):**  
    - Convert raw HTML to clean markdown text for AI consumption  
    - Connect DATA FOR SEO1 → Markdown1

22. **Add Set Nodes ("Edit Fields31", "Edit Fields34"):**  
    - Store raw HTML response for AI prompt construction  
    - Set error messages for scraping failures  
    - Connect Markdown1 → Edit Fields31 and Edit Fields34

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                               | Context or Link                                                                        |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------|
| Workflow combines DataForSEO real-time scraping and SEO metrics API with Claude AI via OpenRouter for deep business and SEO analysis, producing detailed HTML reports converted to PDF and emailed automatically.                                                                                                                               | Sticky Note12: Automated SEO Intelligence Platform description                         |
| DataForSEO API usage may be rate-limited; large websites can take 2-5 minutes to process fully. Consider delays or plan upgrades for bulk processing.                                                                                                                                                                                        | Sticky Note13: API Rate Limits!                                                       |
| AI agents generate business research and product strategy insights forming the foundation of the SEO audit report. The report follows a strict structure with multiple sections such as Executive Summary, Business Summary, Target Audience, Content Architecture, Technical SEO, EEAT, Local SEO, AI Search Optimization, Conversion Audit, KPIs, and Website Insights. | Sticky Note11: AI Analysis & Strategic Insights                                      |
| DataForSEO API documentation and usage details: https://dataforseo.com/                                                                                                                                                                                                                                                                      | Official DataForSEO API website                                                        |
| OpenRouter platform for Claude and Gemini LLMs: https://openrouter.ai/                                                                                                                                                                                                                                                                      | OpenRouter website                                                                     |
| PDF.co API for PDF generation: https://pdf.co/                                                                                                                                                                                                                                                                                              | PDF.co website                                                                         |
| LinkedIn profile of workflow author for support and inquiries: https://www.linkedin.com/in/harsh-agrawal-490138256/                                                                                                                                                                                                                         | Sticky Note12: LinkedIn contact link                                                  |

---

**Disclaimer:**  
The provided content is exclusively from an automated workflow created with n8n, adhering strictly to current content policies. It contains no illegal, offensive, or protected elements. All data handled is legal and publicly available.