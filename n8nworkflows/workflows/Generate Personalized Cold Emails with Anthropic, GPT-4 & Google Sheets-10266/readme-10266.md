Generate Personalized Cold Emails with Anthropic, GPT-4 & Google Sheets

https://n8nworkflows.xyz/workflows/generate-personalized-cold-emails-with-anthropic--gpt-4---google-sheets-10266


# Generate Personalized Cold Emails with Anthropic, GPT-4 & Google Sheets

---
### 1. Workflow Overview

This workflow automates the generation of personalized cold emails by integrating lead data from Google Sheets with AI-powered content synthesis using Anthropic Claude and OpenAI GPT-4. It is designed for sales and marketing teams seeking to scale personalized outreach by extracting meaningful operational insights from a lead’s website and social profiles, then crafting targeted email subject lines and bodies that incorporate a specific sales offer.

The workflow is logically divided into the following functional blocks:

- **1.1 Data Ingestion & Filtering:** Retrieves leads from Google Sheets and filters for completeness and readiness.
- **1.2 Scraping & Data Shaping:** Scrapes lead websites, extracts text and social links, and assesses content quality to determine lead viability.
- **1.3 Filtering Quality Leads:** Uses an IF condition node as a gatekeeper to pass only high-quality leads to AI processing, optimizing cost and efficiency.
- **1.4 AI Personalization Engine:** Uses GPT-4 to summarize website insights, Anthropic Claude to generate cold email subject and body content in JSON format, and code node to parse AI outputs.
- **1.5 Logging, Draft Creation & Notifications:** Logs final results back to Google Sheets, creates Gmail drafts for manual review, loops over leads, and sends a completion alert to the team.

---

### 2. Block-by-Block Analysis

#### 2.1 Data Ingestion & Filtering

**Overview:**  
This block initiates the workflow via manual trigger, loads all leads from a Google Sheet, and filters them to ensure only qualified prospects with valid emails, websites, and empty prior icebreaker fields proceed.

**Nodes Involved:**  
- Manual Trigger  
- Get All Leads (Google Sheets)  
- Filter (Qualified Leads)

**Node Details:**  

- **Manual Trigger**
  - Type: Trigger node to start workflow manually.
  - Configuration: No parameters.
  - Connections: Output to "Get All Leads".
  - Failure cases: User must trigger manually; no automatic start.

- **Get All Leads (Google Sheets)**
  - Type: Google Sheets read node.
  - Configuration: Reads sheet "gid=0" from document ID "1rUf_ReJMP1sQGUVj7Zfj4SYnmmM06_ZIOP1quabD6hI".
  - Credentials: Google Sheets OAuth2.
  - Output: Rows of leads with fields including email, website_url, icebreaker_subject, icebreaker_body, etc.
  - Edge cases: Sheet or ID not found; authentication failure.

- **Filter (Qualified Leads)**
  - Type: Filter node.
  - Configuration: AND conditions requiring:
    - icebreaker_subject empty
    - icebreaker_body empty
    - email not empty
    - website_url not empty
  - Purpose: Only leads missing prior icebreaker data and having necessary contact info proceed.
  - Output: Passes valid leads to "Loop Over Leads".
  - Edge cases: Data inconsistencies or missing fields may exclude leads.

---

#### 2.2 Scraping & Data Shaping

**Overview:**  
Processes each lead individually to scrape their website, extract textual content and social media links, and determine the quality of the scraped data for further AI processing.

**Nodes Involved:**  
- Loop Over Leads (Split In Batches)  
- Capture Lead Data (Set)  
- Scrape Site (HTTP Request)  
- Extract Text & Links (HTML extraction)  
- Filter Social & Status (Code)

**Node Details:**  

- **Loop Over Leads (Split In Batches)**
  - Type: Batch processing node.
  - Configuration: Default batch size (1 by implication).
  - Purpose: Processes each lead individually for error isolation.
  - Input: Filtered leads.
  - Outputs: Two outputs:
    - Main: to "Send Team Completion Alert" (after all batches)
    - Main: to "Capture Lead Data" for each lead.
  - Failure: Batch size issues or node crashes affect lead processing isolation.

- **Capture Lead Data (Set)**
  - Type: Set node to structure lead data.
  - Configuration: Extracts and assigns fields like row_number, id, first_name, last_name, email, website_url, title, company from current batch lead.
  - Purpose: Packages lead data for downstream nodes.
  - Output: To "Scrape Site".
  - Edge cases: Missing or malformed lead fields.

- **Scrape Site (HTTP Request)**
  - Type: HTTP request node.
  - Configuration:
    - URL dynamically set to lead’s website_url.
    - Response format: File (binary) for HTML content.
    - User-Agent header set to mimic a common browser.
  - Purpose: Retrieves the website HTML content.
  - Output: Binary HTML content to "Extract Text & Links".
  - Failure: Website unavailable, timeouts, or blocking by target site.

- **Extract Text & Links (HTML Extraction)**
  - Type: HTML node.
  - Configuration:
    - Extracts all href attributes from anchor tags (links).
    - Extracts body text content.
    - Options trim and clean text.
  - Purpose: Obtains website textual content and all outbound links for analysis.
  - Output: JSON with website_text and links arrays.
  - Failure: Malformed HTML, missing expected elements.

- **Filter Social & Status (Code)**
  - Type: Code node (JavaScript).
  - Configuration:
    - Filters extracted links to keep only social media domains (linkedin.com, twitter.com, instagram.com).
    - Joins social links into a single string.
    - Checks if website text length > 50 characters to set status 'Success' or 'Scrape Fail'.
    - Packages original lead data, social links string, website text, and status.
  - Purpose: Acts as quality gate and enriches lead data.
  - Output: Passes enriched lead data and status to IF node.
  - Failure: Runtime errors in code; missing fields.

---

#### 2.3 Filtering Quality Leads (IF Node)

**Overview:**  
This node uses a conditional check on the status field to either send leads with successful scrapes to AI processing or bypass them directly to logging to save costs.

**Nodes Involved:**  
- If (status equals "Success")

**Node Details:**  

- **If**
  - Type: Conditional node.
  - Configuration: Checks if `$json.status` equals "Success".
  - True branch: Proceeds to "Summarize Website" (OpenAI GPT-4 node).
  - False branch: Bypasses AI nodes and goes to "Log Final Result" to update Google Sheets directly.
  - Purpose: Avoids spending tokens on poor quality leads.
  - Edge cases: Status field missing or malformed leads to misrouting.

---

#### 2.4 AI Personalization Engine

**Overview:**  
This block synthesizes the lead’s website data into a concise company summary using GPT-4, then generates a personalized cold email subject and body using Anthropic Claude. The output is parsed into clean strings for final use.

**Nodes Involved:**  
- Summarize Website (OpenAI GPT-4)  
- Merge Summary (Set)  
- Generate Subject & Body (Anthropic Claude)  
- Parse AI Output (Code)

**Node Details:**  

- **Summarize Website (OpenAI GPT-4)**
  - Type: LangChain OpenAI node.
  - Configuration:
    - Model: GPT-4 (chatgpt-4o-latest).
    - Temperature: 0.1 (low for analytical consistency).
    - System prompt instructs:
      - Paragraph 1: Summarize company’s core value proposition.
      - Paragraph 2: Extract unique operational insight related to customer interaction burdens.
    - Inputs: Website text and social links.
  - Output: Company summary text.
  - Credentials: OpenAI API.
  - Failure: API rate limits, malformed input, or incomplete data.

- **Merge Summary (Set)**
  - Type: Set node.
  - Configuration: Packages the website summary and the original lead data into one object for Anthropic input.
  - Purpose: Prepares context for next AI prompt.
  - Output: To "Generate Subject & Body".

- **Generate Subject & Body (Anthropic Claude)**
  - Type: LangChain Anthropic node.
  - Configuration:
    - Model: claude-haiku-4-5-20251001.
    - Temperature: 0.
    - System prompt defines persona as a Spartan cold email copywriter.
    - Requires output as a strict JSON object with "subject" and "body" fields.
    - Incorporates company summary, lead company name, and a specific sales offer (Free 48-Hour Pilot).
    - Input variables include lead name and company summary.
  - Credentials: Anthropic API.
  - Output: Nested JSON string wrapped in markdown.
  - Failure: Parsing errors, API errors, or incorrect prompt format.

- **Parse AI Output (Code)**
  - Type: Code node.
  - Configuration:
    - Strips markdown code block wrappers.
    - Parses JSON from Anthropic output.
    - Extracts "subject" and "body" fields.
    - Returns clean strings along with original lead data and website summary.
  - Failure: JSON parse exceptions, malformed AI output.

---

#### 2.5 Logging, Draft Creation & Notifications

**Overview:**  
Finalizes the workflow by updating the Google Sheet with generated emails and status, creating Gmail drafts for review, looping over leads, and sending a completion alert to the team.

**Nodes Involved:**  
- Log Final Result (Google Sheets update)  
- Create a draft (Gmail draft creation)  
- Loop Over Leads (batch looping)  
- Send Team Completion Alert (Gmail)

**Node Details:**  

- **Log Final Result (Google Sheets)**
  - Type: Google Sheets update node.
  - Configuration:
    - Matches rows by lead "id".
    - Updates lead row with new status, generated icebreaker_subject, icebreaker_body, and other lead info.
    - Handles both Success and Scrape Fail paths.
  - Credentials: Google Sheets OAuth2.
  - Edge cases: Sheet write permission errors, ID mismatches.

- **Create a draft (Gmail)**
  - Type: Gmail node creating draft emails.
  - Configuration:
    - Composes email with subject and body from AI output.
    - Sends draft to lead’s email address.
    - Saves as draft, does not send immediately.
  - Credentials: Gmail OAuth2.
  - Purpose: Allows manual quality control before sending.
  - Failure: Gmail auth errors, formatting issues.

- **Loop Over Leads (main loop)**
  - As described earlier, loops back to process next lead after draft creation.
  - Output "After Last Batch" goes to "Send Team Completion Alert".

- **Send Team Completion Alert (Gmail)**
  - Type: Gmail node.
  - Configuration:
    - Sends a simple alert email to the workflow owner upon completion of all batches.
    - Notifies that personalized cold emails are ready in Gmail drafts.
  - Credentials: Gmail OAuth2.
  - Failure: Email delivery errors.

---

### 3. Summary Table

| Node Name                 | Node Type                    | Functional Role                                         | Input Node(s)                  | Output Node(s)                       | Sticky Note                                                                                                                     |
|---------------------------|------------------------------|---------------------------------------------------------|--------------------------------|------------------------------------|--------------------------------------------------------------------------------------------------------------------------------|
| Manual Trigger            | manualTrigger                | Initiates workflow manually                              | -                              | Get All Leads                      |                                                                                                                                 |
| Get All Leads             | googleSheets                 | Reads lead data from Google Sheets                       | Manual Trigger                 | Filter (Qualified Leads)           | ## **1. Data Ingestion & Filtering** ... Replace the placeholder Spreadsheet ID and Sheet Name with your master lead sheet details. |
| Filter (Qualified Leads)  | filter                      | Filters leads for valid email, website, empty icebreaker | Get All Leads                  | Loop Over Leads                   | ## **1. Data Ingestion & Filtering** ... Uses AND conditions to ensure prospect readiness.                                      |
| Loop Over Leads           | splitInBatches              | Processes leads one by one in batches                    | Filter (Qualified Leads)        | Send Team Completion Alert, Capture Lead Data | ## **5. Logging, Draft and Alert** ... Loop-back after draft creation for next lead.                                        |
| Capture Lead Data         | set                         | Structures lead data for scraping                        | Loop Over Leads                | Scrape Site                      |                                                                                                                                 |
| Scrape Site               | httpRequest                 | Retrieves website HTML content                           | Capture Lead Data              | Extract Text & Links             | ## **2. Scraping & Data Shaping** ... Response format set to File for binary data.                                              |
| Extract Text & Links      | html                        | Extracts website text and all links                      | Scrape Site                   | Filter Social & Status           | ## **2. Scraping & Data Shaping** ... Simultaneous extraction of text and links.                                                |
| Filter Social & Status    | code                        | Filters social links, assigns status based on text length| Extract Text & Links           | If                             | ## **2. Scraping & Data Shaping** ... Filters social links and sets status Success or Scrape Fail.                               |
| If                       | if                          | Routes leads based on scraping success                   | Filter Social & Status         | Summarize Website (True), Log Final Result (False) | ## **3. IF (Filtering)** ... Acts as quality gate to save token costs on bad leads.                                         |
| Summarize Website         | langchain.openAi            | Uses GPT-4 to summarize company profile and insight     | If (True branch)               | Merge Summary                   | ## **4. AI Personalization Engine** ... GPT-4 extracts unique operational insight and core value proposition.                 |
| Merge Summary             | set                         | Packages summary and lead data for Anthropic prompt     | Summarize Website             | Generate Subject & Body          | ## **4. AI Personalization Engine** ... Prepares context for Anthropic.                                                         |
| Generate Subject & Body   | langchain.anthropic         | Generates cold email subject and body in JSON format    | Merge Summary                 | Parse AI Output                 | ## **4. AI Personalization Engine** ... Claude generates cohesive email content with strict JSON output.                       |
| Parse AI Output           | code                        | Parses and cleans Anthropic JSON output                  | Generate Subject & Body       | Log Final Result                | ## **4. AI Personalization Engine** ... Parses markdown and extracts clean subject and body.                                    |
| Log Final Result          | googleSheets                | Updates Google Sheet with final status and email content| Parse AI Output, If (False)    | Create a draft                 | ## **5. Logging, Draft and Alert** ... Logs both success and scrape fail leads back to sheet.                                   |
| Create a draft            | gmail                       | Creates Gmail draft email for manual review              | Log Final Result              | Loop Over Leads                | ## **5. Logging, Draft and Alert** ... Sends draft email to Gmail for QA before sending.                                        |
| Send Team Completion Alert| gmail                       | Sends notification email to workflow owner on completion| Loop Over Leads (After Last Batch) | -                          | ## **5. Logging, Draft and Alert** ... Alerts team that personalized emails are ready in Gmail drafts.                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node** to start the workflow manually.

2. **Add a Google Sheets node ("Get All Leads")**:
   - Operation: Read Rows.
   - Document ID: Set your master lead spreadsheet ID.
   - Sheet Name: Use the sheet containing leads (e.g., "gid=0").
   - Credentials: Set up Google Sheets OAuth2 credential with access to the sheet.

3. **Add a Filter node ("Filter (Qualified Leads)")**:
   - Configure AND conditions:
     - icebreaker_subject is empty
     - icebreaker_body is empty
     - email is not empty
     - website_url is not empty
   - Connect "Get All Leads" output to this node.

4. **Add a SplitInBatches node ("Loop Over Leads")**:
   - Default batch size (1).
   - Connect Filter's output to this node.

5. **Add a Set node ("Capture Lead Data")**:
   - Set fields: row_number, id, first_name, last_name, email, website_url, title, company.
   - Use expressions to map from current item JSON.
   - Connect "Loop Over Leads" output to this node.

6. **Add an HTTP Request node ("Scrape Site")**:
   - URL: Set to `={{ $json.website_url }}`.
   - Response Format: File (binary).
   - Headers: Add User-Agent mimicking a browser.
   - Connect "Capture Lead Data" to this node.

7. **Add an HTML Extract node ("Extract Text & Links")**:
   - Operation: Extract HTML Content.
   - Source Data: Binary.
   - Extraction Values:
     - links: Extract `href` attribute from all `<a>` tags, return as array.
     - website_text: Extract text content from `<body>`.
   - Connect "Scrape Site" output to this node.

8. **Add a Code node ("Filter Social & Status")**:
   - Paste provided JavaScript code to:
     - Filter social links (linkedin.com, twitter.com, instagram.com).
     - Determine status "Success" if website_text length > 50, else "Scrape Fail".
     - Package lead data for downstream nodes.
   - Connect "Extract Text & Links" output here.

9. **Add an IF node ("If")**:
   - Condition: `$json.status` equals "Success".
   - True branch leads to "Summarize Website".
   - False branch leads directly to "Log Final Result".
   - Connect "Filter Social & Status" output here.

10. **Add a LangChain OpenAI node ("Summarize Website")**:
    - Model: chatgpt-4o-latest (GPT-4).
    - Temperature: 0.1.
    - System prompt:
      - Summarize company value proposition and unique operational insight.
    - Input: Pass website_text and social_links from previous node.
    - Credentials: Set OpenAI API key.
    - Connect True output of "If" here.

11. **Add a Set node ("Merge Summary")**:
    - Assign website_summary from GPT-4 output.
    - Pass along all lead data from previous nodes.
    - Connect "Summarize Website" output here.

12. **Add a LangChain Anthropic node ("Generate Subject & Body")**:
    - Model: claude-haiku-4-5-20251001.
    - Temperature: 0.
    - System prompt defines a Spartan cold email copywriter persona with strict JSON output format.
    - Input variables: company name, website summary, recipient name.
    - Credentials: Anthropic API key.
    - Connect "Merge Summary" output here.

13. **Add a Code node ("Parse AI Output")**:
    - JavaScript to strip markdown wrappers and parse JSON.
    - Extract subject and body fields cleanly.
    - Connect output of Anthropic node here.

14. **Add a Google Sheets node ("Log Final Result")**:
    - Operation: Update Row.
    - Match rows by "id".
    - Update columns: id, email, status, last_name, first_name, row_number, icebreaker_body, Icebreaker_subject.
    - Credentials: Google Sheets OAuth2.
    - Connect output of "Parse AI Output" to this node.
    - Also connect False output of "If" node directly here to log scrape failures.

15. **Add a Gmail node ("Create a draft")**:
    - Operation: Create Draft.
    - To: Lead’s email.
    - Subject: Use parsed subject.
    - Message: Use parsed body.
    - Credentials: Gmail OAuth2.
    - Connect output of "Log Final Result" here.

16. **Connect "Create a draft" output back to "Loop Over Leads"**:
    - This loops the workflow for the next lead.

17. **Connect the "After Last Batch" output of "Loop Over Leads" to Gmail node ("Send Team Completion Alert")**:
    - Configure to send an email to your team notifying that drafts are ready.
    - Credentials: Gmail OAuth2.

---

### 5. General Notes & Resources

| Note Content                                                                                                                       | Context or Link                                                               |
|-----------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------|
| Workflow uses a strict JSON output format from Anthropic Claude to ensure clean parsing of AI-generated email content.            | See Anthropic node prompt in block 2.4.                                      |
| The IF node acts as a critical cost-saving gate by filtering out low-quality leads before invoking expensive AI nodes.           | Sticky Note3 explains this design choice.                                    |
| The scraping uses a User-Agent header to reduce blocking by target websites.                                                      | Sticky Note4 details HTTP request configuration.                             |
| The final draft emails are saved in Gmail drafts for manual QA before sending, reducing risk of AI hallucination or formatting errors. | Sticky Note2 describes the logging and draft creation process.               |
| For support or questions about this workflow, connect on LinkedIn: [https://www.linkedin.com/in/bhuvaneshhhh/](https://www.linkedin.com/in/bhuvaneshhhh/) | Sticky Note12.                                                               |

---

**Disclaimer:**  
The provided text is exclusively from an automated workflow created with n8n, an integration and automation tool. This process strictly respects current content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and publicly available.