Extract LinkedIn Profiles with Apify & Auto-Generate Lead Data for Google Sheets CRM

https://n8nworkflows.xyz/workflows/extract-linkedin-profiles-with-apify---auto-generate-lead-data-for-google-sheets-crm-6116


# Extract LinkedIn Profiles with Apify & Auto-Generate Lead Data for Google Sheets CRM

### 1. Workflow Overview

This workflow automates the extraction of LinkedIn profile data using Apify, processes and scores the lead information, and then inserts the structured lead data into a Google Sheets CRM. It supports multiple input sources for LinkedIn URLs, including manual trigger, scheduled checks, and webhook calls, consolidating these into a unified processing pipeline. The workflow's logic is divided into the following blocks:

- **1.1 Input Reception:** Collect LinkedIn profile URLs from various sources — manual trigger, scheduled trigger for batch processing, webhook input, and Google Sheets.
- **1.2 Input Processing & Formatting:** Normalize and format the incoming URLs into a consistent batch for processing.
- **1.3 Batch Handling & Splitting:** Determine if URLs require batch processing and split accordingly.
- **1.4 LinkedIn Profile Scraping:** Call Apify or an HTTP request node to scrape LinkedIn profile data from the URLs.
- **1.5 Profile Data Processing & Scoring:** Transform raw scraped data into structured lead data and apply scoring logic.
- **1.6 Routing & CRM Update:** Decide whether to update Google Sheets CRM and perform the insertion.
- **1.7 Response & Completion:** Respond to webhook calls with success messages and finalize the workflow execution.

Sticky notes in the workflow provide contextual business value, feature explanations, scoring logic, usage examples, and configuration notes.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
This block collects LinkedIn URLs from multiple sources to feed into the lead generation pipeline. It supports manual, scheduled, webhook, and Google Sheets inputs for flexibility.

**Nodes Involved:**  
- Manual Trigger  
- Check New Entries (Schedule Trigger)  
- Webhook Input  
- Read LinkedIn URLs (Google Sheets)  

**Node Details:**

- **Manual Trigger**  
  - Type: Trigger  
  - Role: Allows manual execution of the workflow for ad hoc runs or testing.  
  - Configuration: Default, no parameters needed.  
  - Input: None  
  - Output: Passes empty data to "Process All Input Sources".  
  - Edge Cases: None expected.  

- **Check New Entries (Schedule Trigger)**  
  - Type: Trigger  
  - Role: Periodically initiates the workflow to check for new LinkedIn URLs from Google Sheets.  
  - Configuration: Default schedule settings (user-defined in n8n interface).  
  - Input: None  
  - Output: Triggers "Read LinkedIn URLs".  
  - Edge Cases: Missed schedule runs, rate limiting if run too frequently.  

- **Webhook Input**  
  - Type: Webhook  
  - Role: Receives LinkedIn URLs or lead data via HTTP POST calls from external systems.  
  - Configuration: Webhook ID "linkedin-webhook-001" uniquely identifies this webhook.  
  - Input: HTTP request payload.  
  - Output: Passes data to "Process All Input Sources".  
  - Edge Cases: Authentication or security not detailed; potential for invalid input.  

- **Read LinkedIn URLs (Google Sheets)**  
  - Type: Google Sheets node  
  - Role: Reads LinkedIn URLs stored in a Google Sheets document to feed into processing.  
  - Configuration: Google Sheets credentials required; sheet and range specified by user.  
  - Input: Triggered by "Check New Entries".  
  - Output: Passes rows to "Process All Input Sources".  
  - Edge Cases: Permissions, connectivity, empty or malformed sheets.  

---

#### 2.2 Input Processing & Formatting

**Overview:**  
Consolidates inputs from all sources into a uniform format. This prepares URLs for batch processing by normalizing input data structures.

**Nodes Involved:**  
- Process All Input Sources (Code)  
- Format Input Data (Set)  
- Check Batch Processing (If)

**Node Details:**

- **Process All Input Sources (Code)**  
  - Type: Function (Code)  
  - Role: Consolidates and cleans input from manual, webhook, and Google Sheets into a consistent array of LinkedIn URLs.  
  - Configuration: Custom JavaScript code to parse and merge inputs.  
  - Expressions: Likely uses `$input` and `$json` to access incoming data.  
  - Inputs: Data from "Webhook Input", "Manual Trigger", and "Read LinkedIn URLs".  
  - Outputs: Standardized data for "Format Input Data".  
  - Edge Cases: Null or malformed data entries, inconsistent formats.  

- **Format Input Data (Set)**  
  - Type: Set node  
  - Role: Sets and formats key fields such as URL lists, batch size, or metadata for further processing.  
  - Configuration: Defines fields explicitly to ensure uniformity.  
  - Input: Data from "Process All Input Sources".  
  - Output: Passes formatted data to "Check Batch Processing".  
  - Edge Cases: Missing fields or incorrect data types.  

- **Check Batch Processing (If)**  
  - Type: If condition  
  - Role: Determines whether the input should be processed in batches (multiple URLs) or individually.  
  - Configuration: Condition based on length of URL list or batch size parameter.  
  - Input: From "Format Input Data".  
  - Outputs:  
    - True: to "Split Batch URLs" for batch processing  
    - False: directly to "Scrape LinkedIn Profile" for single URL processing.  
  - Edge Cases: Empty URL list, incorrect batch size.  

---

#### 2.3 Batch Handling & Splitting

**Overview:**  
If multiple LinkedIn URLs are detected, this block splits the batch into individual URLs for sequential processing.

**Nodes Involved:**  
- Split Batch URLs (Code)

**Node Details:**

- **Split Batch URLs (Code)**  
  - Type: Function (Code)  
  - Role: Divides the list of URLs into individual entries for scraping.  
  - Configuration: Custom JavaScript splitting an array into single-item objects.  
  - Input: Batch URLs from "Check Batch Processing".  
  - Output: Each URL sent to "Scrape LinkedIn Profile" node.  
  - Edge Cases: Empty array, malformed URLs.  

---

#### 2.4 LinkedIn Profile Scraping

**Overview:**  
This block scrapes LinkedIn profile data by sending HTTP requests to Apify or an equivalent scraping service.

**Nodes Involved:**  
- Scrape LinkedIn Profile (HTTP Request)

**Node Details:**

- **Scrape LinkedIn Profile**  
  - Type: HTTP Request  
  - Role: Calls external API (likely Apify) to scrape LinkedIn profile data using URLs.  
  - Configuration:  
    - Endpoint URL configured to Apify LinkedIn scraper API.  
    - Method: Typically GET or POST depending on API.  
    - Authentication via API key or headers.  
    - Payload includes LinkedIn profile URL.  
  - Input: Individual URLs from "Split Batch URLs" or directly from "Check Batch Processing" if single URL.  
  - Output: Raw scraped profile data to "Process Profile Data".  
  - Edge Cases: API rate limits, network failures, invalid or private LinkedIn profiles, authentication errors.  

---

#### 2.5 Profile Data Processing & Scoring

**Overview:**  
Transforms raw scraped data into structured lead profiles with additional scoring logic to prioritize leads.

**Nodes Involved:**  
- Process Profile Data (Code)  
- Route to Google Sheets (If)

**Node Details:**

- **Process Profile Data (Code)**  
  - Type: Function (Code)  
  - Role: Parses raw JSON profile data, extracts relevant fields, computes lead scores based on characteristics (e.g., job title, location).  
  - Configuration: Custom JavaScript includes scoring algorithms and data normalization.  
  - Input: Raw data from "Scrape LinkedIn Profile".  
  - Output: Structured lead objects for conditional routing.  
  - Edge Cases: Incomplete data, unexpected JSON structure, scoring logic errors.  

- **Route to Google Sheets (If)**  
  - Type: If condition  
  - Role: Evaluates whether the processed lead meets criteria to be added to Google Sheets CRM (e.g., minimum score).  
  - Configuration: Condition on scoring field or completeness of lead data.  
  - Input: Processed lead from "Process Profile Data".  
  - Outputs:  
    - True: to "Add to Google Sheets CRM"  
    - False: leads may be discarded or logged (no output node connected here).  
  - Edge Cases: Missing score, false negatives.  

---

#### 2.6 Routing & CRM Update

**Overview:**  
Adds qualified leads to Google Sheets CRM and responds to webhook calls confirming success.

**Nodes Involved:**  
- Add to Google Sheets CRM (Google Sheets)  
- Respond Success (Respond to Webhook)

**Node Details:**

- **Add to Google Sheets CRM**  
  - Type: Google Sheets node  
  - Role: Inserts or updates lead data into the CRM spreadsheet.  
  - Configuration:  
    - Google Sheets credentials required.  
    - Target spreadsheet and sheet specified.  
    - Append mode to add new rows.  
  - Input: Qualified leads from "Route to Google Sheets".  
  - Output: Passes success status to "Respond Success".  
  - Edge Cases: Sheet access errors, duplicate entries, API limits.  

- **Respond Success**  
  - Type: Respond to Webhook  
  - Role: Sends HTTP success response to the original webhook caller when processing completes.  
  - Configuration: Custom response content, typically HTTP 200 with confirmation message.  
  - Input: From "Add to Google Sheets CRM".  
  - Output: Workflow end.  
  - Edge Cases: Late response due to long processing, webhook timeout at caller side.  

---

#### 2.7 Sticky Notes (Documentation & Context)

**Overview:**  
Provides business value, feature highlights, scoring logic explanation, usage examples, and configuration notes to aid users and maintainers.

**Nodes Involved:**  
- Business Value  
- Workflow Info  
- Features & Config  
- Scoring Logic  
- Usage Examples  

**Node Details:**

- All are sticky notes (n8n-nodes-base.stickyNote) containing textual explanations positioned near relevant workflow sections.  
- No direct input/output connections.  
- Useful for understanding rationale and configuration hints.  
- Edge Cases: None  

---

### 3. Summary Table

| Node Name               | Node Type               | Functional Role                     | Input Node(s)                | Output Node(s)                  | Sticky Note                                          |
|-------------------------|-------------------------|-----------------------------------|-----------------------------|-------------------------------|-----------------------------------------------------|
| Manual Trigger          | Trigger                 | Manual workflow start             | None                        | Process All Input Sources      |                                                     |
| Check New Entries       | Schedule Trigger        | Scheduled workflow start          | None                        | Read LinkedIn URLs             |                                                     |
| Webhook Input           | Webhook                 | Receive LinkedIn URLs externally  | None                        | Process All Input Sources      |                                                     |
| Read LinkedIn URLs      | Google Sheets           | Read URLs from Google Sheets      | Check New Entries            | Process All Input Sources      |                                                     |
| Process All Input Sources| Function (Code)        | Consolidate all input sources     | Manual Trigger, Webhook Input, Read LinkedIn URLs | Format Input Data        |                                                     |
| Format Input Data       | Set                     | Format and standardize inputs     | Process All Input Sources    | Check Batch Processing         |                                                     |
| Check Batch Processing  | If                      | Determine batch or single URL     | Format Input Data            | Split Batch URLs, Scrape LinkedIn Profile |                                                     |
| Split Batch URLs        | Function (Code)          | Split batch into individual URLs  | Check Batch Processing (true) | Scrape LinkedIn Profile        |                                                     |
| Scrape LinkedIn Profile | HTTP Request            | Scrape LinkedIn data via API      | Split Batch URLs, Check Batch Processing (false) | Process Profile Data      |                                                     |
| Process Profile Data    | Function (Code)          | Process and score profile data    | Scrape LinkedIn Profile      | Route to Google Sheets         |                                                     |
| Route to Google Sheets  | If                      | Decide if lead qualifies for CRM  | Process Profile Data         | Add to Google Sheets CRM       |                                                     |
| Add to Google Sheets CRM| Google Sheets           | Add lead to CRM spreadsheet       | Route to Google Sheets       | Respond Success                |                                                     |
| Respond Success         | Respond to Webhook      | Respond to webhook HTTP caller    | Add to Google Sheets CRM     | None                          |                                                     |
| Business Value          | Sticky Note             | Business context and value        | None                        | None                          |                                                     |
| Workflow Info           | Sticky Note             | Workflow description and metadata | None                        | None                          |                                                     |
| Features & Config       | Sticky Note             | Features explanation & config     | None                        | None                          |                                                     |
| Scoring Logic           | Sticky Note             | Lead scoring explanation          | None                        | None                          |                                                     |
| Usage Examples          | Sticky Note             | Example usage scenarios            | None                        | None                          |                                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the "Manual Trigger" node:**  
   - Type: Manual Trigger  
   - No configuration needed.  

2. **Create the "Check New Entries" node:**  
   - Type: Schedule Trigger  
   - Configure schedule (e.g., every hour/day) as desired.  

3. **Create the "Webhook Input" node:**  
   - Type: Webhook  
   - Set Webhook ID to "linkedin-webhook-001" (or custom).  
   - Configure authentication/security as needed.  

4. **Create the "Read LinkedIn URLs" node:**  
   - Type: Google Sheets  
   - Set credentials for Google Sheets OAuth2.  
   - Configure to read from the spreadsheet and range containing LinkedIn URLs.  

5. **Create the "Process All Input Sources" node:**  
   - Type: Function (Code)  
   - Add JavaScript code to merge inputs from Manual Trigger, Webhook Input, and Google Sheets into a flat array of URLs.  
   - Use `$input.all()` or equivalent to access multiple inputs.  

6. **Create the "Format Input Data" node:**  
   - Type: Set  
   - Define fields such as `urls` (array), `batchSize` (optional integer), and any metadata.  
   - Map outputs from previous function node.  

7. **Create the "Check Batch Processing" node:**  
   - Type: If  
   - Condition: Check if `urls.length > 1` or batchSize > 1  
   - True branch: connect to "Split Batch URLs"  
   - False branch: connect to "Scrape LinkedIn Profile"  

8. **Create the "Split Batch URLs" node:**  
   - Type: Function (Code)  
   - Code to iterate over `urls` array and output each URL as separate item.  

9. **Create the "Scrape LinkedIn Profile" node:**  
   - Type: HTTP Request  
   - Configure HTTP method and URL for Apify LinkedIn scraper API.  
   - Add authentication headers or API key from credentials.  
   - Pass LinkedIn URL in request body or query parameters.  

10. **Create the "Process Profile Data" node:**  
    - Type: Function (Code)  
    - JavaScript code to parse API response, extract lead fields (name, title, location, etc.), and calculate lead score.  

11. **Create the "Route to Google Sheets" node:**  
    - Type: If  
    - Condition: Check if lead score exceeds threshold.  
    - True branch: connect to "Add to Google Sheets CRM".  

12. **Create the "Add to Google Sheets CRM" node:**  
    - Type: Google Sheets  
    - Use OAuth2 credentials with write permission.  
    - Configure to append new rows in the CRM sheet.  
    - Map lead fields to columns accordingly.  

13. **Create the "Respond Success" node:**  
    - Type: Respond to Webhook  
    - Configure to send HTTP 200 with confirmation message.  
    - Connect it after "Add to Google Sheets CRM".  

14. **Connect triggers (Manual Trigger, Schedule Trigger, Webhook Input) to "Process All Input Sources".**  
15. **Connect "Process All Input Sources" → "Format Input Data" → "Check Batch Processing".**  
16. **Connect "Check Batch Processing" true → "Split Batch URLs" → "Scrape LinkedIn Profile".**  
17. **Connect "Check Batch Processing" false → "Scrape LinkedIn Profile".**  
18. **Connect "Scrape LinkedIn Profile" → "Process Profile Data" → "Route to Google Sheets".**  
19. **Connect "Route to Google Sheets" true → "Add to Google Sheets CRM" → "Respond Success".**

20. **Add sticky notes for documentation:**  
    - Business Value near triggers.  
    - Workflow Info near start.  
    - Features & Config near input processing.  
    - Scoring Logic near profile processing.  
    - Usage Examples near CRM nodes.  

---

### 5. General Notes & Resources

| Note Content                                                                                   | Context or Link                                  |
|-----------------------------------------------------------------------------------------------|-------------------------------------------------|
| Workflow automates LinkedIn lead extraction with Apify and integrates to Google Sheets CRM.    | Workflow purpose and overview                    |
| Scoring logic is customizable in the "Process Profile Data" node to prioritize leads.          | Critical for lead qualification                   |
| Use Google Sheets OAuth2 credentials with appropriate sheet access for reading and writing.    | Credential setup                                 |
| Webhook ID "linkedin-webhook-001" must be unique; secure webhook endpoint recommended.         | Security best practice                            |
| Apify LinkedIn scraper requires API key and may have usage limits; monitor API quota.          | Integration constraints                           |
| Consider error handling additions for network failures, API limits, and data validation.       | Workflow robustness                              |
| For usage examples and advanced configurations, refer to sticky notes inside the workflow.     | In-workflow documentation                         |

---

**Disclaimer:** The provided content is generated exclusively from an automated n8n workflow. It adheres strictly to content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly accessible.