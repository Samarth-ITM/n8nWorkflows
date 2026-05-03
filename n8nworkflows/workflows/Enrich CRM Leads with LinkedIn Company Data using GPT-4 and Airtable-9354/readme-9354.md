Enrich CRM Leads with LinkedIn Company Data using GPT-4 and Airtable

https://n8nworkflows.xyz/workflows/enrich-crm-leads-with-linkedin-company-data-using-gpt-4-and-airtable-9354


# Enrich CRM Leads with LinkedIn Company Data using GPT-4 and Airtable

### 1. Workflow Overview

This workflow automates the enrichment of CRM lead records with detailed company data extracted from LinkedIn company profiles, leveraging AI (GPT-4) for advanced analysis and structured data extraction. It is designed primarily for sales, marketing, and business development teams who use Airtable as their CRM.

**Target Use Cases:**  
- Automatically enhancing lead records in Airtable with rich, structured company information from LinkedIn.  
- Generating email-ready personalized variables for outreach campaigns.  
- Maintaining up-to-date, enriched data to improve outreach effectiveness.

**Logical Blocks:**

- **1.1 Input Reception and Lead Retrieval:** Fetches a lead record from Airtable CRM, including the LinkedIn company URL.  
- **1.2 LinkedIn Data Acquisition:** Scrapes the raw HTML content of the LinkedIn company profile page.  
- **1.3 Data Cleaning:** Cleans the scraped HTML to produce formatted plain text suitable for AI processing.  
- **1.4 AI Company Profile Analysis:** Uses GPT-4 to analyze and extract a structured company report from the cleaned LinkedIn profile text.  
- **1.5 Email Variable Extraction:** Converts the AI-generated company report into personalized, email-ready variables.  
- **1.6 CRM Update:** Updates the original Airtable lead record with the enriched company data and marks the lead as enriched.

---

### 2. Block-by-Block Analysis

#### 2.1 Block 1: Input Reception and Lead Retrieval

- **Overview:** Retrieves lead data from Airtable CRM, including the LinkedIn company URL needed for subsequent enrichment steps.

- **Nodes Involved:**  
  - Fetch Lead from CRM  
  - Step 1 Note (sticky note)  

- **Node Details:**

  - **Fetch Lead from CRM**  
    - **Type:** Airtable node  
    - **Role:** Retrieves a single lead record by ID from Airtable CRM base and table specified.  
    - **Configuration:**  
      - Uses dynamic record ID input `={{ $json.recordId }}` to fetch the specific lead.  
      - Requires user to replace `YOUR_AIRTABLE_BASE_ID` and `YOUR_TABLE_ID` with their actual Airtable base and table IDs.  
    - **Input:** Trigger or prior workflow parameter supplying record ID.  
    - **Output:** JSON containing lead data including LinkedIn company URL.  
    - **Failure Modes:** Possible auth errors if Airtable credentials are invalid; errors if recordId is missing or incorrect; rate limits if overused.  
    - **Version:** Airtable node version 2.1.

  - **Step 1 Note**  
    - Sticky note summarizing the step purpose.

#### 2.2 Block 2: LinkedIn Data Acquisition

- **Overview:** Scrapes the LinkedIn company profile HTML content using the URL from lead data.

- **Nodes Involved:**  
  - Scrape LinkedIn Company Profile  
  - Step 2 Note (sticky note)

- **Node Details:**

  - **Scrape LinkedIn Company Profile**  
    - **Type:** HTTP Request node  
    - **Role:** Performs GET request to the LinkedIn company URL to retrieve raw HTML content.  
    - **Configuration:**  
      - URL is dynamically set from the lead's LinkedIn Organization URL field (e.g., `={{ $json['LinkedIn Organization URL'] }}`).  
      - No special headers or authentication configured (may need adjustments for LinkedIn anti-scraping mechanisms).  
    - **Input:** Output from "Fetch Lead from CRM".  
    - **Output:** Raw HTML content under `.data`.  
    - **Failure Modes:** Request failures due to LinkedIn blocking scraping, rate limiting, invalid URL, or network issues.  
    - **Version:** HTTP Request node version 4.2.

  - **Step 2 Note**  
    - Sticky note explaining the purpose of scraping LinkedIn profile.

#### 2.3 Block 3: Data Cleaning

- **Overview:** Cleans the raw HTML scraped from LinkedIn by removing scripts, styles, and HTML tags to produce clean, readable plain text for AI input.

- **Nodes Involved:**  
  - Clean HTML Content  
  - Step 3 Note (sticky note)

- **Node Details:**

  - **Clean HTML Content**  
    - **Type:** Code node (JavaScript)  
    - **Role:** Processes the scraped HTML to remove tags and decode entities for clean text extraction.  
    - **Configuration:**  
      - Runs once per item (`mode: runOnceForEachItem`).  
      - JavaScript code removes `<script>`, `<style>` tags, strips all HTML tags, decodes common HTML entities (`&nbsp;`, `&amp;`, etc.), and normalizes whitespace.  
      - Returns a single field `plainText` containing cleaned text.  
    - **Input:** Raw HTML content from HTTP Request node.  
    - **Output:** Plain text string ready for AI analysis.  
    - **Failure Modes:** If HTML content is missing or malformed, output may be empty or inaccurate. Expression errors if node referencing is incorrect.  
    - **Version:** Code node version 2.

  - **Step 3 Note**  
    - Sticky note describing the cleaning process.

#### 2.4 Block 4: AI Company Profile Analysis

- **Overview:** Uses GPT-4 AI model to analyze the cleaned LinkedIn profile text and generate a structured, detailed company report with multiple labeled sections.

- **Nodes Involved:**  
  - Analyze Company Profile with AI  
  - Step 4 Note (sticky note)

- **Node Details:**

  - **Analyze Company Profile with AI**  
    - **Type:** OpenAI node (via Langchain integration)  
    - **Role:** Sends plain text to GPT-4 model with a detailed system prompt instructing to produce a 5-section company LinkedIn report.  
    - **Configuration:**  
      - Model: `gpt-4o-mini` (GPT-4 optimized variant).  
      - System prompt defines role, task, output structure (5 labeled sections), data extraction rules (exact employee count, recent posts with timestamps).  
      - Input message content is the cleaned plain text.  
      - Output is JSON-parsed response containing the structured report under the "Company LinkedIn" string.  
    - **Input:** Cleaned plain text from prior code node.  
    - **Output:** Structured company profile text with overview, products, operations, announcements, and company posts.  
    - **Failure Modes:** API rate limits, invalid API key, malformed input, or model timeouts. Output formatting errors if AI response deviates from expected structure.  
    - **Version:** OpenAI node version 1.8.

  - **Step 4 Note**  
    - Sticky note summarizing AI analysis step.

#### 2.5 Block 5: Email Variable Extraction

- **Overview:** Converts the AI-generated company report text into specific, formatted variables ready for personalized email templates.

- **Nodes Involved:**  
  - Extract Email-Ready Variables  
  - Step 5 Note (sticky note)

- **Node Details:**

  - **Extract Email-Ready Variables**  
    - **Type:** OpenAI node (Langchain integration)  
    - **Role:** Parses the company profile text to extract precise variables such as company name, industry, employee count, mission statement, products, funding details, hiring status, etc., all formatted naturally for insertion in outreach emails.  
    - **Configuration:**  
      - Model: `gpt-4o-mini`.  
      - System prompt details extraction rules for each variable, specifying fallback to `"not listed"` if data is missing, and proper sentence casing.  
      - Input message includes the AI analysis output plus current date for contextual reference.  
      - Returns a JSON object with all email variables.  
    - **Input:** Output from AI company profile analysis node.  
    - **Output:** JSON object with email-ready variables.  
    - **Failure Modes:** API errors, incomplete AI output, or deviation from prompt structure.  
    - **Version:** OpenAI node version 1.8.

  - **Step 5 Note**  
    - Sticky note explaining the extraction of email variables.

#### 2.6 Block 6: CRM Update

- **Overview:** Writes the enriched company data back to the Airtable CRM record, marking the lead as enriched and ready for outreach.

- **Nodes Involved:**  
  - Update CRM with Enriched Data  
  - Step 6 Note (sticky note)

- **Node Details:**

  - **Update CRM with Enriched Data**  
    - **Type:** Airtable node  
    - **Role:** Updates the lead record in Airtable by setting fields such as Lead Status, Date Lead Enriched, and clearing the enrichment trigger flag.  
    - **Configuration:**  
      - Uses the record ID from the original fetched lead.  
      - Sets "Lead Status" to `"Cold"` (possibly a placeholder or default).  
      - Clears "LinkedIn Company" field (set as `=` which likely clears or resets).  
      - Sets "Date Lead Enriched" to current date in ISO format.  
      - Sets "Start Lead Enrichment" field to `false` to avoid re-triggering.  
      - Requires Airtable base and table IDs to be configured by user.  
    - **Input:** Email-ready variables are not directly mapped here, but presumably manual mapping or automation outside this node integrates them.  
    - **Output:** Confirmation of updated record.  
    - **Failure Modes:** Auth errors, invalid record ID, concurrency conflicts, or network errors.  
    - **Version:** Airtable node version 2.1.

  - **Step 6 Note**  
    - Sticky note describing update operation.

#### Additional Node: Email Variables Reference

- **Type:** Sticky Note  
- **Role:** Provides detailed documentation of all email template variables generated in step 5, with example usage snippets for outreach emails.  
- **Position:** Separate from main flow, serves as user-facing documentation.

---

### 3. Summary Table

| Node Name                      | Node Type                         | Functional Role                                      | Input Node(s)               | Output Node(s)                 | Sticky Note                                                                                                                              |
|--------------------------------|----------------------------------|-----------------------------------------------------|-----------------------------|-------------------------------|------------------------------------------------------------------------------------------------------------------------------------------|
| Workflow Description            | Sticky Note                      | Provides high-level workflow overview and setup     | -                           | -                             | ## 🚀 Enrich CRM Leads with LinkedIn Company Data Using AI ... (full description as above)                                               |
| Step 1 Note                    | Sticky Note                      | Describes Step 1: retrieving lead data              | -                           | -                             | ## 📋 Step 1: Retrieve Lead Data ... (fetches lead from Airtable)                                                                         |
| Fetch Lead from CRM            | Airtable                        | Fetches lead record from Airtable by record ID      | (trigger or param)           | Scrape LinkedIn Company Profile |                                                                                                                                          |
| Step 2 Note                    | Sticky Note                      | Describes Step 2: scraping LinkedIn profile         | -                           | -                             | ## 🌐 Step 2: Scrape LinkedIn Profile ... (retrieves raw HTML)                                                                            |
| Scrape LinkedIn Company Profile| HTTP Request                    | Retrieves LinkedIn company profile HTML              | Fetch Lead from CRM          | Clean HTML Content             |                                                                                                                                          |
| Step 3 Note                    | Sticky Note                      | Describes Step 3: cleaning HTML content              | -                           | -                             | ## 🧹 Step 3: Clean HTML Content ... (removes tags, scripts, styles)                                                                      |
| Clean HTML Content             | Code (JavaScript)               | Cleans HTML to produce plain text for AI             | Scrape LinkedIn Company Profile | Analyze Company Profile with AI |                                                                                                                                          |
| Step 4 Note                    | Sticky Note                      | Describes Step 4: AI company analysis                 | -                           | -                             | ## 🤖 Step 4: AI Company Analysis ... (GPT-4 extraction)                                                                                  |
| Analyze Company Profile with AI| OpenAI (Langchain)              | Analyzes cleaned text to produce structured report   | Clean HTML Content           | Extract Email-Ready Variables  |                                                                                                                                          |
| Step 5 Note                    | Sticky Note                      | Describes Step 5: extracting email variables         | -                           | -                             | ## 📊 Step 5: Extract Email Variables ... (email-ready formatting)                                                                        |
| Extract Email-Ready Variables  | OpenAI (Langchain)              | Parses AI report to extract personalization variables| Analyze Company Profile with AI | Update CRM with Enriched Data |                                                                                                                                          |
| Step 6 Note                    | Sticky Note                      | Describes Step 6: updating CRM with enriched data    | -                           | -                             | ## 💾 Step 6: Update CRM Record ... (write back to Airtable)                                                                              |
| Update CRM with Enriched Data  | Airtable                       | Updates lead record with enriched data                | Extract Email-Ready Variables | -                             |                                                                                                                                          |
| Email Variables Reference      | Sticky Note                      | Provides email template variable documentation        | -                           | -                             | ## 📧 Email Template Variables Reference ... (examples and variable list)                                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Sticky Notes for Documentation:**  
   - Add sticky notes titled "Workflow Description," "Step 1 Note" through "Step 6 Note," and "Email Variables Reference" with content as described in section 1 and 2.

2. **Create Node: Fetch Lead from CRM**  
   - Node Type: Airtable (version 2.1)  
   - Configure credentials for Airtable account.  
   - Set Base ID to your Airtable CRM base.  
   - Set Table ID to your leads table.  
   - Set operation to "Get Record" by ID.  
   - Use expression `={{ $json.recordId }}` for record ID input (replace with your actual method of passing record IDs).  
   - Connect this node as the starting point or trigger input.

3. **Create Node: Scrape LinkedIn Company Profile**  
   - Node Type: HTTP Request (version 4.2)  
   - Configure no authentication (adjust if LinkedIn requires).  
   - Set URL to expression `={{ $json['LinkedIn Organization URL'] }}` to dynamically get URL from lead data.  
   - Set method to GET.  
   - Connect input to "Fetch Lead from CRM" node output.

4. **Create Node: Clean HTML Content**  
   - Node Type: Code (JavaScript) (version 2)  
   - Set mode to "Run Once For Each Item."  
   - Paste the provided JavaScript code that removes `<script>`, `<style>`, strips HTML tags, decodes entities, and normalizes whitespace.  
   - Input: connect from "Scrape LinkedIn Company Profile."  
   - Output: returns a field `plainText`.

5. **Create Node: Analyze Company Profile with AI**  
   - Node Type: OpenAI (Langchain integration) (version 1.8)  
   - Configure OpenAI API credentials with GPT-4 model access.  
   - Select model `gpt-4o-mini` or equivalent GPT-4 model.  
   - Set message with system prompt explaining the role and detailed 5-section output structure (copy prompt exactly).  
   - Set input message content to `={{ $json.plainText }}` from prior node.  
   - Enable JSON output parsing.  
   - Connect input from "Clean HTML Content."

6. **Create Node: Extract Email-Ready Variables**  
   - Node Type: OpenAI (Langchain integration) (version 1.8)  
   - Use same OpenAI credentials as above.  
   - Model: `gpt-4o-mini`.  
   - System prompt instructs extraction of specific email-friendly variables with fallback to `"not listed"`.  
   - Input message content uses the AI company profile output: `={{ $json.message.content }}` plus current date.  
   - Enable JSON output parsing.  
   - Connect input from "Analyze Company Profile with AI."

7. **Create Node: Update CRM with Enriched Data**  
   - Node Type: Airtable (version 2.1)  
   - Use same Airtable credentials as in step 2.  
   - Set Base and Table IDs to match your CRM.  
   - Operation: Update Record.  
   - Use record ID from the first node: `={{ $('Fetch Lead from CRM').first().json.id }}`.  
   - Define fields to update:  
     - Lead Status: `"Cold"` (or your desired status)  
     - LinkedIn Company: clear or set as needed.  
     - Date Lead Enriched: `={{ new Date().toISOString().split('T')[0] }}` (current date)  
     - Start Lead Enrichment: `false` to avoid re-running enrichment.  
   - Connect input from "Extract Email-Ready Variables."

8. **Connect Nodes Sequentially:**  
   - Fetch Lead from CRM → Scrape LinkedIn Company Profile → Clean HTML Content → Analyze Company Profile with AI → Extract Email-Ready Variables → Update CRM with Enriched Data.

9. **Final Setup:**  
   - Add a trigger node as needed (Webhook, Schedule, or Manual) to start the workflow with a lead record ID.  
   - Ensure all credentials (Airtable, OpenAI) are properly configured and tested.  
   - Replace placeholder Base and Table IDs with actual Airtable identifiers.  
   - Confirm LinkedIn URLs are correctly stored in the CRM leads for scraping.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                               | Context or Link                                                                                       |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| Pro tip: Run this workflow on new leads automatically to always have enriched data ready for outreach.                                                                                                                                     | Workflow Description sticky note                                                                     |
| Customize AI prompts to extract different company attributes or adjust variables for your specific email templates.                                                                                                                       | Workflow Description sticky note                                                                     |
| Add additional enrichment sources such as company websites or Crunchbase for deeper insights.                                                                                                                                              | Workflow Description sticky note                                                                     |
| Connect this workflow to other CRMs like HubSpot or Salesforce by adapting Airtable nodes or using dedicated CRM nodes.                                                                                                                  | Workflow Description sticky note                                                                     |
| Example outreach email variable usage provided in the "Email Variables Reference" sticky note for easy template integration.                                                                                                              | Email Variables Reference sticky note                                                                |
| LinkedIn scraping may require proxy or authenticated requests to avoid anti-scraping blocks or CAPTCHAs. Consider enhancing HTTP request node configuration accordingly.                                                                  | Scrape LinkedIn Company Profile node note (implicit)                                                |
| AI output formatting depends heavily on prompt engineering; maintain the prompt carefully to ensure consistent, structured results.                                                                                                       | Analyze Company Profile with AI node note                                                           |
| Monitor API usage and consider rate limits or costs associated with OpenAI GPT-4 usage.                                                                                                                                                | General best practice                                                                                 |

---

**Disclaimer:** The provided text and workflow are generated exclusively from an automated process using n8n integration and automation tools. It complies strictly with content policies and contains no illegal, offensive, or protected material. All data processed is legal and publicly available.