Discover & Enrich Decision-Makers with Apollo and Human Verification

https://n8nworkflows.xyz/workflows/discover---enrich-decision-makers-with-apollo-and-human-verification-3830


# Discover & Enrich Decision-Makers with Apollo and Human Verification

### 1. Workflow Overview

This workflow automates the discovery and enrichment of key decision-maker contacts from a list of companies, integrating Apollo APIs, Google Sheets, Slack, and an LLM service (e.g., OpenAI). It is designed for sales and marketing teams to streamline lead generation by:

- Extracting company data and websites from input company names.
- Enriching company details and generating concise summaries.
- Searching for decision-makers with specific seniority levels at these companies.
- Enriching decision-maker contacts with verified emails and phone numbers.
- Classifying contacts’ departments using an LLM based on job titles.
- Updating a structured leads database in Google Sheets.
- Incorporating human verification via Slack for company websites.
- Sending weekly Slack reports summarizing newly verified leads.

The workflow is logically divided into these blocks:

- **1.1 Input Reception & Filtering:** Trigger and filter new or updated companies needing processing.
- **1.2 Company Data Enrichment:** Retrieve and enrich company details using Apollo APIs and create summaries.
- **1.3 Human Verification:** Request human approval via Slack to confirm or correct company websites.
- **1.4 Decision Maker Discovery:** Search Apollo for key decision-makers at companies using batched API calls.
- **1.5 Contact Processing & Department Classification:** Process individual contacts, classify their departments via LLM, and update the contacts list.
- **1.6 Contact Enrichment:** Bulk enrich contacts to retrieve emails and phone numbers, update contacts sheet.
- **1.7 Verified Contacts Filtering & Reporting:** Filter verified contacts and send a weekly report via Slack.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Filtering

- **Overview:**  
  This block triggers the workflow when changes occur in the Companies tab of a Google Sheet. It filters out companies already processed to avoid duplicate processing.

- **Nodes Involved:**  
  - Google Sheets Trigger  
  - Select Unprocessed Companies (Filter)  
  - Is Domain Already Provided? (If)  
  - Merge Rows Which Now All Contain Domains  

- **Node Details:**

  - **Google Sheets Trigger**  
    - Type: Trigger node, fires on changes in Google Sheets.  
    - Configuration: Polls the `Companies` tab (gid=0) every few minutes.  
    - Inputs: N/A (trigger node)  
    - Outputs: Rows changed in Companies sheet.  
    - Edge cases: Google API quota limits; changes not detected if polling interval too long; incorrect sheet ID causes failure.

  - **Select Unprocessed Companies**  
    - Type: Filter node, filters companies where `Status` ≠ "Processed".  
    - Configuration: Case-sensitive, strict string comparison on the `Status` field.  
    - Inputs: Output of Google Sheets Trigger.  
    - Outputs: Only companies needing processing.  
    - Failure modes: Missing `Status` field or inconsistent casing.

  - **Is Domain Already Provided?**  
    - Type: If node, checks if company `Domain` field is non-empty.  
    - Configuration: Checks if `Domain` is not empty string.  
    - Inputs: Filtered companies.  
    - Outputs:  
      - True branch: Companies with domain -> merge node for further processing.  
      - False branch: Companies without domain -> Apollo Organization Search.  
    - Edge cases: Domain field may include malformed URLs; empty but non-null values.

  - **Merge Rows Which Now All Contain Domains**  
    - Type: Merge node to combine companies with domains from both branches.  
    - Inputs:  
      - From True branch of If (companies with domain).  
      - From Slack approval (companies edited with domain).  
    - Outputs: Companies ready for enrichment.  

---

#### 1.2 Company Data Enrichment

- **Overview:**  
  Enrich company data using Apollo’s Organization Enrichment API, generate a concise business summary via LLM, and update the Companies sheet with enriched data and status.

- **Nodes Involved:**  
  - Apollo Organization Enrichment (HTTP Request)  
  - Summarize Core Business (LLM OpenAI)  
  - Add Company Details (Google Sheets)  

- **Node Details:**

  - **Apollo Organization Enrichment**  
    - Type: HTTP Request node calling Apollo Org Enrich API.  
    - Configuration: Sends `domain` extracted from company domain field, authenticates with `x-api-key`.  
    - Inputs: Companies merged with domains.  
    - Outputs: Detailed company data (name, website, social links, funding, employees, description).  
    - Edge cases: API key invalid, domain not found, network timeout.

  - **Summarize Core Business**  
    - Type: LLM node (OpenAI GPT-4o-mini).  
    - Configuration: Prompt asks to summarize company description into one line.  
    - Inputs: Company description field from API response.  
    - Outputs: Single-line summary of core business.  
    - Edge cases: Empty or malformed description, LLM timeout or rate limit.

  - **Add Company Details**  
    - Type: Google Sheets node, appends or updates Companies tab.  
    - Configuration: Maps enriched fields (phone, domain, social URLs, industry, revenue, size, funding, description), includes generated summary, sets `Status` to "Processed".  
    - Inputs: Combined data from enrichment and summary.  
    - Outputs: Updates Companies sheet.  
    - Edge cases: Google Sheets API limits, schema mismatch, concurrent updates.

---

#### 1.3 Human Verification

- **Overview:**  
  Since website data may be inaccurate, this block sends a Slack message prompting users to verify or correct website URLs in the Google Sheet.

- **Nodes Involved:**  
  - Add Company Website (Google Sheets)  
  - Approve Company Website (Slack Send & Wait)  
  - Merge Rows Which Now All Contain Domains (Merge)  

- **Node Details:**

  - **Add Company Website**  
    - Type: Google Sheets node, updates Companies tab with website and domain fetched from Apollo Organization Search.  
    - Inputs: Apollo Organization Search API response.  
    - Outputs: Updates Companies tab with website details.  
    - Edge cases: Partial data from API, updates overwritten by manual edits.

  - **Approve Company Website**  
    - Type: Slack node, sends message and waits for user approval.  
    - Configuration: Sends message to specific Slack channel with instructions and Google Sheet link.  
    - Inputs: Updated Companies rows.  
    - Outputs: Waits for manual confirmation.  
    - Edge cases: Slack API rate limits, no user response, auth failure.

  - **Merge Rows Which Now All Contain Domains**  
    - See above in Block 1.1.

---

#### 1.4 Decision Maker Discovery

- **Overview:**  
  Uses Apollo People Search API to find decision-makers at the companies. Supports batching of up to 1,000 domains per request with a dynamically constructed URL.

- **Nodes Involved:**  
  - Create Apollo People Search URL (Code)  
  - Apollo Find Decision Makers (HTTP Request)  
  - Loop Over Items (1000 per Batch)  
  - Split Out Batched Decision Maker Response (Split Out)  

- **Node Details:**

  - **Create Apollo People Search URL**  
    - Type: Code node, builds full Apollo People Search API URL including query parameters for seniorities and domains.  
    - Inputs: Batch of company domains.  
    - Outputs: URL string for search request.  
    - Edge cases: Exceeding parameter length limits, encoding issues.

  - **Apollo Find Decision Makers**  
    - Type: HTTP Request node, POSTs to generated search URL.  
    - Configuration: Authenticated with `x-api-key`, JSON headers.  
    - Inputs: URL from Code node.  
    - Outputs: JSON response with decision makers.  
    - Edge cases: API errors, empty responses, rate limits.

  - **Loop Over Items (1000 per Batch)**  
    - Type: SplitInBatches node, batches input companies into groups of 1000 for API calls.  
    - Inputs: Companies with domains.  
    - Outputs: Batches forwarded to URL creation and API call.  
    - Edge cases: Large datasets, batch size tuning.

  - **Split Out Batched Decision Maker Response**  
    - Type: SplitOut node, splits the array of people in the API response into individual items.  
    - Inputs: Batched API response.  
    - Outputs: Individual decision-maker entries.  
    - Edge cases: Empty or malformed response arrays.

---

#### 1.5 Contact Processing & Department Classification

- **Overview:**  
  For each decision-maker, determine their department using an LLM based on job title and update the Contacts tab accordingly.

- **Nodes Involved:**  
  - Determine Contact's Department (LLM OpenAI)  
  - Add Contacts (Google Sheets)  

- **Node Details:**

  - **Determine Contact's Department**  
    - Type: LLM node (OpenAI GPT-4o-mini).  
    - Configuration: Prompt asks to identify department from job title, returning department name only.  
    - Inputs: Individual decision-maker's job title.  
    - Outputs: Department classification string.  
    - Edge cases: Ambiguous or missing titles, LLM failures.

  - **Add Contacts**  
    - Type: Google Sheets node, appends or updates Contacts tab.  
    - Configuration: Maps rich contact info (name, job title, company, location, LinkedIn, department classification, etc.) with upsert using LinkedIn URL as unique key.  
    - Inputs: Decision maker data enriched with department classification.  
    - Outputs: Updated Contacts sheet.  
    - Edge cases: Duplicate LinkedIn URLs, data inconsistencies.

---

#### 1.6 Contact Enrichment

- **Overview:**  
  Since Apollo People Search may not provide verified emails or phones, this block enriches contacts in batches (10 per batch) via Apollo Bulk People Enrichment API, then updates the Contacts tab with verified info.

- **Nodes Involved:**  
  - Loop Over Items For Bulk Enrichment (10 per batch)  
  - Create Apollo People Enrichment Payload (Code)  
  - Apollo Enrich Decision Makers (HTTP Request)  
  - Split Out Batched Enrichment Response (Split Out)  
  - Enrich Contacts (Google Sheets)  

- **Node Details:**

  - **Loop Over Items For Bulk Enrichment (10 per batch)**  
    - Type: SplitInBatches node, batches contacts into groups of 10 for bulk enrichment.  
    - Inputs: Contacts from previous block.  
    - Outputs: Batches forwarded for payload creation.  
    - Edge cases: Batch size constraints, API limits.

  - **Create Apollo People Enrichment Payload**  
    - Type: Code node, formats batch into payload with `linkedin_url` and `domain` fields required by Apollo bulk match API.  
    - Inputs: Batch of contacts.  
    - Outputs: JSON payload for API call.  
    - Edge cases: Missing LinkedIn URLs or domains.

  - **Apollo Enrich Decision Makers**  
    - Type: HTTP Request node, POSTs bulk enrichment payload to Apollo API.  
    - Configuration: Authenticated with `x-api-key`.  
    - Inputs: JSON payload.  
    - Outputs: Bulk enrichment response with emails and phones.  
    - Edge cases: Partial matches, API rate limits.

  - **Split Out Batched Enrichment Response**  
    - Type: SplitOut node, splits array of enriched contact matches into individual items.  
    - Inputs: Bulk enrichment API response.  
    - Outputs: Individual enriched contact records.  
    - Edge cases: Empty or partial data.

  - **Enrich Contacts**  
    - Type: Google Sheets node, updates Contacts tab with enriched emails, verification status, and LinkedIn URL.  
    - Configuration: Upsert using LinkedIn URL.  
    - Inputs: Enriched contacts.  
    - Outputs: Updated Contacts sheet.  
    - Edge cases: Data overwrite risks, sheet API limits.

---

#### 1.7 Verified Contacts Filtering & Reporting

- **Overview:**  
  Filters contacts with verified emails and sends a weekly Slack report summarizing new verified leads discovered in the past week.

- **Nodes Involved:**  
  - Weekly Report Trigger (Schedule Trigger)  
  - Retrieve Verified Leads (Google Sheets)  
  - Derive Past Week's Lead Gen. Metrics (Code)  
  - Send Weekly Report (Slack)  

- **Node Details:**

  - **Weekly Report Trigger**  
    - Type: Schedule Trigger, configured to run weekly on a specific day (Friday).  
    - Inputs: None.  
    - Outputs: Initiates report generation.  
    - Edge cases: Time zone issues, missed runs.

  - **Retrieve Verified Leads**  
    - Type: Google Sheets node, reads Contacts (Verified) tab filtering on `Email Verified = Yes`.  
    - Inputs: Trigger from schedule.  
    - Outputs: List of verified contacts.  
    - Edge cases: Empty data, filter errors.

  - **Derive Past Week's Lead Gen. Metrics**  
    - Type: Code node, calculates unique companies and contacts from retrieved data, formats a summary message.  
    - Inputs: Verified contacts.  
    - Outputs: Summary message JSON.  
    - Edge cases: Incorrect data formats, empty inputs.

  - **Send Weekly Report**  
    - Type: Slack node, sends summary message to configured Slack channel.  
    - Configuration: Uses OAuth2 authentication, includes link to workflow.  
    - Inputs: Summary message.  
    - Outputs: Slack message post.  
    - Edge cases: Slack API errors, auth failures.

---

### 3. Summary Table

| Node Name                           | Node Type                 | Functional Role                                          | Input Node(s)                             | Output Node(s)                              | Sticky Note                                                                                              |
|-----------------------------------|---------------------------|----------------------------------------------------------|------------------------------------------|----------------------------------------------|--------------------------------------------------------------------------------------------------------|
| Google Sheets Trigger             | Google Sheets Trigger     | Trigger workflow on company sheet changes                | N/A                                      | Select Unprocessed Companies                  | Workflow trigger on Google Sheet Companies tab. See sticky note "Workflow Trigger".                     |
| Select Unprocessed Companies      | Filter                   | Filter companies not marked as Processed                  | Google Sheets Trigger                    | Is Domain Already Provided?                    | Status management to avoid duplicates. See sticky note "Status Management".                             |
| Is Domain Already Provided?       | If                       | Check if domain field is present                           | Select Unprocessed Companies             | Merge Rows Which Now All Contain Domains, Apollo Organization Search |                                                                                                        |
| Merge Rows Which Now All Contain Domains | Merge                    | Combine companies with domains from different branches    | Is Domain Already Provided?, Approve Company Website | Apollo Organization Enrichment               |                                                                                                        |
| Apollo Organization Search        | HTTP Request             | Search companies by name if domain missing                 | Is Domain Already Provided? (False branch) | Add Company Website                          |                                                                                                        |
| Add Company Website               | Google Sheets            | Update Companies sheet with website and domain             | Apollo Organization Search               | Approve Company Website, Merge Rows Which Now All Contain Domains |                                                                                                        |
| Approve Company Website           | Slack                    | Send Slack message for human verification of websites     | Add Company Website                      | Merge Rows Which Now All Contain Domains       | Human-in-the-loop verification for websites. See sticky note "Human-in-the-Loop Verification".         |
| Apollo Organization Enrichment    | HTTP Request             | Enrich company details by domain                           | Merge Rows Which Now All Contain Domains | Summarize Core Business                       | Company enrichment using Apollo API. See sticky note "Company Enrichment".                             |
| Summarize Core Business           | LLM (OpenAI)             | Generate one-line summary of company business              | Apollo Organization Enrichment           | Add Company Details                           |                                                                                                        |
| Add Company Details              | Google Sheets            | Update Companies sheet with enriched data and summary      | Summarize Core Business                  | Apollo Find Decision Makers                   |                                                                                                        |
| Create Apollo People Search URL   | Code                     | Build Apollo People Search URL with seniorities and domains | Loop Over Items (1000 per Batch)         | Apollo Find Decision Makers                   | Decision maker discovery batching explained. See sticky note "Decision Maker Discovery".               |
| Apollo Find Decision Makers       | HTTP Request             | Search for decision-makers with seniorities                | Create Apollo People Search URL          | Loop Over Items (1000 per Batch), Split Out Batched Decision Maker Response |                                                                                                        |
| Loop Over Items (1000 per Batch)  | SplitInBatches           | Batch companies for API calls                               | Apollo Find Decision Makers              | Split Out Batched Decision Maker Response, Create Apollo People Search URL |                                                                                                        |
| Split Out Batched Decision Maker Response | SplitOut                 | Split batched decision maker API response                   | Loop Over Items (1000 per Batch)         | Determine Contact's Department                | Data processing and upsertion. See sticky note "Data Processing and Upsertion".                        |
| Determine Contact's Department    | LLM (OpenAI)             | Classify contact department based on job title             | Split Out Batched Decision Maker Response | Add Contacts                                 |                                                                                                        |
| Add Contacts                     | Google Sheets            | Upsert decision makers into Contacts sheet                  | Determine Contact's Department           | Loop Over Items For Bulk Enrichment (10 per batch) |                                                                                                        |
| Loop Over Items For Bulk Enrichment (10 per batch) | SplitInBatches           | Batch contacts for bulk enrichment API calls                | Add Contacts                            | Create Apollo People Enrichment Payload       | Bulk enrichment batching explained. See sticky note "Contact Enrichment".                             |
| Create Apollo People Enrichment Payload | Code                     | Prepare batch payload with LinkedIn URLs and domains       | Loop Over Items For Bulk Enrichment (10 per batch) | Apollo Enrich Decision Makers               |                                                                                                        |
| Apollo Enrich Decision Makers     | HTTP Request             | Bulk enrich contacts to retrieve emails and phones         | Create Apollo People Enrichment Payload  | Split Out Batched Enrichment Response          |                                                                                                        |
| Split Out Batched Enrichment Response | SplitOut                 | Split batched enrichment response into individual contacts  | Apollo Enrich Decision Makers            | Enrich Contacts                              |                                                                                                        |
| Enrich Contacts                 | Google Sheets            | Update Contacts sheet with enriched email and verification  | Split Out Batched Enrichment Response    | N/A                                          | Verified contacts filtering note. See sticky note "Verified Contacts Filtering".                       |
| Weekly Report Trigger            | Schedule Trigger         | Trigger weekly report generation                            | N/A                                      | Retrieve Verified Leads                       | Weekly report automation. See sticky note "Weekly Report".                                            |
| Retrieve Verified Leads          | Google Sheets            | Retrieve contacts with verified emails                      | Weekly Report Trigger                    | Derive Past Week's Lead Gen. Metrics          |                                                                                                        |
| Derive Past Week's Lead Gen. Metrics | Code                     | Calculate unique companies and contacts, format message    | Retrieve Verified Leads                  | Send Weekly Report                            |                                                                                                        |
| Send Weekly Report              | Slack                    | Post weekly summary message to Slack channel                | Derive Past Week's Lead Gen. Metrics    | N/A                                          |                                                                                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Sheets Trigger node**  
   - Poll Companies tab (`gid=0`) of the specified Google Sheet every few minutes to detect row additions or updates.

2. **Add Filter node "Select Unprocessed Companies"**  
   - Pass only rows where `Status` ≠ "Processed".

3. **Add If node "Is Domain Already Provided?"**  
   - Condition: `Domain` field is not empty.

4. **Add Merge node "Merge Rows Which Now All Contain Domains"**  
   - To combine companies with domains from both branches.

5. **Add HTTP Request node "Apollo Organization Search"**  
   - POST to `https://api.apollo.io/api/v1/mixed_companies/search` with query parameter `q_organization_name` set to company name.  
   - Include `x-api-key` header with Apollo API key.

6. **Add Google Sheets node "Add Company Website"**  
   - Update Companies sheet with website and domain fields from Apollo Organization Search response.  
   - Set matching column to `Domain`.

7. **Add Slack node "Approve Company Website"**  
   - Send message to configured Slack channel requesting human verification of company website.  
   - Use OAuth2 authentication and enable "send and wait" to pause for user input.

8. **Connect "Approve Company Website" output back to "Merge Rows Which Now All Contain Domains"**.

9. **Connect merged companies with domains to HTTP Request node "Apollo Organization Enrichment"**  
   - POST to `https://api.apollo.io/api/v1/organizations/enrich` with `domain` query parameter.  
   - Include required headers with API key.

10. **Add LLM node "Summarize Core Business" (OpenAI GPT-4o-mini)**  
    - Prompt: Summarize company description in one line.

11. **Add Google Sheets node "Add Company Details"**  
    - Append or update Companies tab with enriched details and summary.  
    - Set `Status` to "Processed".

12. **Add Code node "Create Apollo People Search URL"**  
    - Generate Apollo People Search API URL with seniorities and batch domains (up to 1000 per batch).

13. **Add SplitInBatches node "Loop Over Items (1000 per Batch)"**  
    - Batch companies into groups of 1000.

14. **Add HTTP Request node "Apollo Find Decision Makers"**  
    - POST to generated People Search URL with API key headers.

15. **Add SplitOut node "Split Out Batched Decision Maker Response"**  
    - Split people array from API response into single items.

16. **Add LLM node "Determine Contact's Department"**  
    - Prompt: Determine department name from job title.

17. **Add Google Sheets node "Add Contacts"**  
    - Upsert contacts into Contacts sheet using LinkedIn Profile URL as unique key.  
    - Include department classification and other contact details.

18. **Add SplitInBatches node "Loop Over Items For Bulk Enrichment (10 per batch)"**  
    - Batch contacts into groups of 10.

19. **Add Code node "Create Apollo People Enrichment Payload"**  
    - Format batch payload with LinkedIn URLs and domains.

20. **Add HTTP Request node "Apollo Enrich Decision Makers"**  
    - POST bulk enrichment payload to Apollo bulk_match endpoint with API key.

21. **Add SplitOut node "Split Out Batched Enrichment Response"**  
    - Split matches array into individual enriched contacts.

22. **Add Google Sheets node "Enrich Contacts"**  
    - Update Contacts sheet with enriched emails, verification status, etc., upserting by LinkedIn URL.

23. **Add Schedule Trigger node "Weekly Report Trigger"**  
    - Set to trigger weekly on Friday.

24. **Add Google Sheets node "Retrieve Verified Leads"**  
    - Get rows from Contacts (Verified) tab where `Email Verified = Yes`.

25. **Add Code node "Derive Past Week's Lead Gen. Metrics"**  
    - Calculate unique companies and contacts, generate summary message.

26. **Add Slack node "Send Weekly Report"**  
    - Post summary message to Slack channel with OAuth2 auth.

27. **Connect all nodes as described in the connections section above.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                          | Context or Link                                                                                   |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| This workflow processes companies and outputs curated decision-makers with contact info into a structured leads database. It sends weekly reports via Slack. The Google Sheet uses Apps Script to auto-update status on website/domain edits.           | [Google Sheet Template](https://docs.google.com/spreadsheets/d/1KqKFZ7Uxrt1MivBjLklGdPRMgNBhEc0slpthoSjt2wI/edit?gid=0) |
| The main workflow trigger is adding or updating a row in the Companies tab. The domain is automatically extracted or retrieved via Apollo if missing.                                                                                               | Sticky note "Workflow Trigger" with image link in workflow.                                     |
| Status management uses a filter to only process companies where Status ≠ "Processed". Editing website resets status to "Pending" via Apps Script to enable reprocessing.                                                                              | Sticky note "Status Management".                                                                |
| Company enrichment uses Apollo Organization Enrichment API and an LLM to generate concise company summaries, updating the Companies sheet and setting Status to "Processed".                                                                           | Sticky note "Company Enrichment".                                                               |
| Decision maker discovery uses Apollo People Search API with batching and dynamic URL creation to handle up to 1000 domains per request.                                                                                                             | Sticky note "Decision Maker Discovery".                                                         |
| Processing includes splitting batched API responses, classifying contacts’ departments using an LLM, and upserting contacts into Google Sheets.                                                                                                     | Sticky note "Data Processing and Upsertion".                                                   |
| Contact enrichment involves bulk enrichment via Apollo Bulk People Enrichment API in batches of 10 to retrieve emails and phone numbers, followed by updating the Contacts sheet again.                                                              | Sticky note "Contact Enrichment".                                                              |
| Contacts with verified emails are filtered in the Contacts (Verified) tab using a spreadsheet formula. A weekly Slack report summarizing new verified leads is automatically sent out.                                                               | Sticky notes "Verified Contacts Filtering" and "Weekly Report".                                 |
| Human verification via Slack is required for confirming or correcting company website URLs to ensure accuracy before further processing.                                                                                                            | Sticky note "Human-in-the-Loop Verification".                                                  |

---

This comprehensive reference enables understanding, reproduction, and modification of the workflow while anticipating common failure cases, credential requirements, and integration nuances.