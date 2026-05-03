Deduplicate Scraping AI Grants for Eligibility using AI

https://n8nworkflows.xyz/workflows/deduplicate-scraping-ai-grants-for-eligibility-using-ai-2619


# Deduplicate Scraping AI Grants for Eligibility using AI

### 1. Workflow Overview

This workflow automates the process of scraping AI-related grant opportunities from grants.gov, filtering out duplicates, qualifying them for eligibility using AI, storing results in Airtable, and sending a daily email alert with a summary of new eligible grants to subscribed team members. It is designed to streamline grant discovery and qualification, reducing manual effort and increasing coverage.

The workflow consists of the following logical blocks:

- **1.1 Scheduled Trigger & Grant Data Fetching:** Periodically fetches the latest AI grant opportunities from grants.gov.
- **1.2 Deduplication of Grants:** Uses n8n’s Remove Duplicates node to filter out grants already processed in previous runs.
- **1.3 Detailed Grant Data Retrieval:** Fetches detailed information for each new grant.
- **1.4 AI-Based Grant Summarization and Eligibility Analysis:** Uses OpenAI language models to summarize grant synopsis and assess eligibility.
- **1.5 Data Aggregation and Storage:** Merges AI outputs and saves grant data with eligibility results to Airtable.
- **1.6 Email Newsletter Generation:** Collects new eligible grants, compiles an HTML email newsletter.
- **1.7 Subscriber Retrieval and Email Sending:** Retrieves active subscribers from Airtable and sends the alert email via Gmail.
- **1.8 Scheduling for Regular Execution:** Two scheduled triggers coordinate fetching and emailing tasks at different times in the morning.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger & Grant Data Fetching

- **Overview:**  
  This block triggers the workflow at 8:30 AM daily to fetch AI-related grant listings posted in the last day from grants.gov.

- **Nodes Involved:**  
  - `Everyday @ 8.30am` (Schedule Trigger)  
  - `AI Grants since Yesterday` (HTTP Request)  
  - `Grants to List` (Split Out)

- **Node Details:**

  - **Everyday @ 8.30am**  
    - Type: Schedule Trigger  
    - Configured to trigger daily at 08:30 AM local time.  
    - No inputs; triggers the workflow.  
    - Outputs trigger to `AI Grants since Yesterday`.

  - **AI Grants since Yesterday**  
    - Type: HTTP Request  
    - Sends POST request to `https://apply07.grants.gov/grantsws/rest/opportunities/search` with JSON body filtering grants by keyword "ai", sorted by open date, posted within 1 day, and status as forecasted or posted.  
    - Returns a JSON with grant opportunities in the field `oppHits`.  
    - Output connected to `Grants to List`.

    **Potential Failures:**  
    - Network errors, API downtime, or unexpected response formats.

  - **Grants to List**  
    - Type: Split Out  
    - Splits the `oppHits` array into individual grant items for downstream processing.  
    - Outputs each grant as a separate item.

---

#### 1.2 Deduplication of Grants

- **Overview:**  
  This block filters out grants that have been processed before by tracking grant IDs across workflow executions, ensuring only new grants proceed.

- **Nodes Involved:**  
  - `Only New Grants` (Remove Duplicates)

- **Node Details:**

  - **Only New Grants**  
    - Type: Remove Duplicates  
    - Configured to remove items seen in previous workflow executions based on the grant’s `id` field.  
    - Ensures deduplication across workflow runs.  
    - Input: individual grants from `Grants to List`.  
    - Output: filtered grants forwarded to `Get Grant Details`.

    **Edge Cases:**  
    - If grant IDs change format or are missing, duplicates might not be correctly identified.  
    - Initial run processes all grants; subsequent runs only new ones.

---

#### 1.3 Detailed Grant Data Retrieval

- **Overview:**  
  Fetches full details for each new grant using the grant ID to enable deeper analysis.

- **Nodes Involved:**  
  - `Get Grant Details` (HTTP Request)

- **Node Details:**

  - **Get Grant Details**  
    - Type: HTTP Request  
    - POST request to `https://apply07.grants.gov/grantsws/rest/opportunity/details` with form-urlencoded body parameter `oppId` set dynamically from grant `id`.  
    - Retrieves detailed JSON object for each grant.  
    - Input: filtered new grants from `Only New Grants`.  
    - Outputs to both `Summarize Synopsis` and `Eligibility Factors` nodes for parallel AI processing.

    **Potential Failures:**  
    - API request failures, timeouts, or invalid IDs causing empty or error responses.

---

#### 1.4 AI-Based Grant Summarization and Eligibility Analysis

- **Overview:**  
  Uses OpenAI’s language model nodes to generate a human-readable summary of the grant and assess eligibility based on company-specific criteria embedded in the system prompt.

- **Nodes Involved:**  
  - `OpenAI Chat Model` (OpenAI LLM)  
  - `Summarize Synopsis` (Information Extractor)  
  - `OpenAI Chat Model1` (OpenAI LLM)  
  - `Eligibility Factors` (Information Extractor)  
  - `Merge` (Merge)

- **Node Details:**

  - **OpenAI Chat Model**  
    - Type: Langchain OpenAI Chat Model  
    - Processes input for summarization.  
    - Credentials: OpenAI API key configured.  
    - Connected as AI language model for `Summarize Synopsis`.

  - **Summarize Synopsis**  
    - Type: Information Extractor  
    - Input text composed from grant agency name, title, and synopsis description.  
    - System prompt requests a simple summary of the opportunity.  
    - Extracts structured output: goal, duration, success criteria, good to know notes.  
    - Receives AI completions from `OpenAI Chat Model`.  
    - Outputs to `Merge` node.

  - **OpenAI Chat Model1**  
    - Type: Langchain OpenAI Chat Model  
    - Processes input for eligibility determination.  
    - Credentials: OpenAI API key configured.  
    - Connected as AI language model for `Eligibility Factors`.

  - **Eligibility Factors**  
    - Type: Information Extractor  
    - Input text includes agency info, title, synopsis, and eligibility description.  
    - System prompt includes detailed company background and instructions to assess eligibility.  
    - Extracts an array of eligibility matches (criteria met).  
    - Receives AI completions from `OpenAI Chat Model1`.  
    - Outputs to `Merge` node.

  - **Merge**  
    - Type: Merge  
    - Combines outputs from `Summarize Synopsis` and `Eligibility Factors` by position, creating a unified record per grant.  
    - Output forwarded to `Save to Tracker`.

    **Edge Cases:**  
    - AI model failures or rate limits may cause incomplete or delayed responses.  
    - Misinterpretation by AI if input data is inconsistent.  
    - Ensure company details in system prompt are accurate for best eligibility results.

---

#### 1.5 Data Aggregation and Storage

- **Overview:**  
  Saves the enriched grant data along with AI-generated summaries and eligibility assessments into an Airtable base for tracking and further manual review.

- **Nodes Involved:**  
  - `Save to Tracker` (Airtable)

- **Node Details:**

  - **Save to Tracker**  
    - Type: Airtable  
    - Operation: Create new records in the specified Airtable base and table (US Grants.gov Tracker).  
    - Maps multiple fields including URL, Title, Agency, Funding, Duration, Eligibility status, Eligibility notes, and more from combined AI outputs and grant details.  
    - Uses Airtable Personal Access Token credential.  
    - Input: merged grant data from `Merge`.

    **Potential Failures:**  
    - Airtable API limits or authentication errors.  
    - Data mapping issues if field names or types change in Airtable.

---

#### 1.6 Email Newsletter Generation

- **Overview:**  
  Retrieves new eligible grants added today, compiles them into a formatted HTML newsletter using a custom email template.

- **Nodes Involved:**  
  - `Everyday @ 9am` (Schedule Trigger)  
  - `Get New Eligible Grants Today` (Airtable Search)  
  - `Generate Email` (HTML Template)

- **Node Details:**

  - **Everyday @ 9am**  
    - Type: Schedule Trigger  
    - Fires daily at 9:00 AM to start the email generation process.

  - **Get New Eligible Grants Today**  
    - Type: Airtable Search  
    - Queries Airtable table for grants with `Status` = 'New' AND `Eligibility?` = 'Yes' AND created date equal to today.  
    - Retrieves only today's newly approved eligible grants.

  - **Generate Email**  
    - Type: HTML Template  
    - Uses a complex HTML email template styled responsively for email clients.  
    - Loops through input grants to render titles, agencies, synopsis goals, success criteria, and dates with links.  
    - Outputs the compiled HTML newsletter content.  
    - Configured to execute once per trigger.

    **Edge Cases:**  
    - If no new eligible grants exist, email content will be empty.  
    - Complex HTML may render differently across email clients.

---

#### 1.7 Subscriber Retrieval and Email Sending

- **Overview:**  
  Fetches a list of active subscribers from Airtable and sends the generated newsletter to each subscriber using Gmail.

- **Nodes Involved:**  
  - `Get Subscribers` (Airtable Search)  
  - `Send Subscriber Email` (Gmail)

- **Node Details:**

  - **Get Subscribers**  
    - Type: Airtable Search  
    - Searches Airtable Subscribers table for records with `Status` = 'Active'.  
    - Executes once per workflow run after email generation.

  - **Send Subscriber Email**  
    - Type: Gmail  
    - Sends an email to each subscriber’s email address with the subject "Daily Newsletter for Interesting US Grants".  
    - Uses the HTML content from `Generate Email`.  
    - Credentials: Gmail OAuth2 configured.  
    - Iterates over subscriber list.

    **Potential Failures:**  
    - Gmail API quota limits or OAuth token expiration.  
    - Invalid email addresses causing delivery failures.

---

#### 1.8 Scheduling for Regular Execution

- **Overview:**  
  Two scheduled triggers coordinate the workflow: one at 8:30 AM to fetch and process grants, and another at 9:00 AM to generate and send the alert email.

- **Nodes Involved:**  
  - `Everyday @ 8.30am`  
  - `Everyday @ 9am`

- **Node Details:**

  - Both are Schedule Trigger nodes with specified daily trigger times.  
  - They serve as entry points for different parts of the workflow to ensure data is processed before email dispatch.

---

### 3. Summary Table

| Node Name               | Node Type                         | Functional Role                                  | Input Node(s)            | Output Node(s)                 | Sticky Note                                                                                                          |
|-------------------------|----------------------------------|-------------------------------------------------|--------------------------|-------------------------------|----------------------------------------------------------------------------------------------------------------------|
| Everyday @ 8.30am       | Schedule Trigger                 | Starts grant fetching process daily at 8:30 AM |                          | AI Grants since Yesterday      | Scheduled triggers run workflow automatically in morning; adjust as needed. [Learn more](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.scheduletrigger) |
| AI Grants since Yesterday| HTTP Request                    | Fetches AI-related grants from grants.gov       | Everyday @ 8.30am        | Grants to List                 |                                                                                                                      |
| Grants to List           | Split Out                      | Splits grant array into individual grants       | AI Grants since Yesterday| Only New Grants                |                                                                                                                      |
| Only New Grants          | Remove Duplicates              | Filters already processed grants                 | Grants to List            | Get Grant Details             | Remove Duplicates tracks grant IDs across executions to avoid duplicates. [Learn more](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.removeduplicates/) |
| Get Grant Details        | HTTP Request                   | Retrieves detailed grant info                     | Only New Grants           | Summarize Synopsis, Eligibility Factors |                                                                                                                      |
| OpenAI Chat Model        | Langchain OpenAI Chat Model    | AI model for summarizing grant synopsis          |                          | Summarize Synopsis (AI input) | Add company details in system prompt for better eligibility results.                                                 |
| Summarize Synopsis       | Information Extractor          | Summarizes grant opportunity                      | Get Grant Details, OpenAI | Merge                        | Uses AI to create simple summary from grant data. [Learn more](https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.information-extractor/) |
| OpenAI Chat Model1       | Langchain OpenAI Chat Model    | AI model for eligibility analysis                 |                          | Eligibility Factors (AI input)| Add company details in system prompt.                                                                                |
| Eligibility Factors      | Information Extractor          | Extracts eligibility matches from grant data     | Get Grant Details, OpenAI1| Merge                        |                                                                                                                      |
| Merge                   | Merge                         | Combines AI outputs into one record               | Summarize Synopsis, Eligibility Factors | Save to Tracker              |                                                                                                                      |
| Save to Tracker          | Airtable                      | Stores grant info with eligibility in Airtable   | Merge                    |                               | Stores grants in Airtable. Sample base: https://airtable.com/appiNoPRvhJxz9crl/shrRdP6zstgsxjDKL. [Learn more](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.airtable/) |
| Everyday @ 9am           | Schedule Trigger               | Triggers email generation daily at 9:00 AM       |                          | Get New Eligible Grants Today |                                                                                                                      |
| Get New Eligible Grants Today | Airtable Search           | Retrieves new eligible grants added today         | Everyday @ 9am           | Generate Email                |                                                                                                                      |
| Generate Email           | HTML Template                 | Generates formatted email newsletter HTML         | Get New Eligible Grants Today | Get Subscribers            | Generates newsletter from grants data. [Learn more](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.html/) |
| Get Subscribers          | Airtable Search              | Retrieves active subscribers for email            | Generate Email           | Send Subscriber Email         |                                                                                                                      |
| Send Subscriber Email    | Gmail                        | Sends the newsletter email to subscribers         | Get Subscribers          |                               | Sends email via Gmail. [Learn more](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.gmail/)         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Scheduled Trigger to Fetch Grants**  
   - Add a **Schedule Trigger** node named `Everyday @ 8.30am`.  
   - Set it to run daily at 8:30 AM.

2. **Fetch AI Grants**  
   - Add an **HTTP Request** node named `AI Grants since Yesterday`.  
   - Set Method: POST.  
   - URL: `https://apply07.grants.gov/grantsws/rest/opportunities/search`.  
   - Request Body Type: JSON.  
   - Body JSON:  
     ```json
     {
       "keyword": "ai",
       "cfda": null,
       "agencies": null,
       "sortBy": "openDate|desc",
       "rows": 5000,
       "eligibilities": null,
       "fundingCategories": null,
       "fundingInstruments": null,
       "dateRange": "1",
       "oppStatuses": "forecasted|posted"
     }
     ```  
   - Connect `Everyday @ 8.30am` output to this node.

3. **Split Grants into Individual Items**  
   - Add a **Split Out** node `Grants to List`.  
   - Set Field to Split Out: `oppHits`.  
   - Connect `AI Grants since Yesterday` output to this node.

4. **Remove Previously Seen Grants**  
   - Add **Remove Duplicates** node `Only New Grants`.  
   - Set Operation: Remove Items Seen in Previous Executions.  
   - Deduplicate By: Expression `{{$json.id}}`.  
   - Connect `Grants to List` output to this node.

5. **Fetch Detailed Grant Information**  
   - Add **HTTP Request** node `Get Grant Details`.  
   - Method: POST.  
   - URL: `https://apply07.grants.gov/grantsws/rest/opportunity/details`.  
   - Content Type: Form-URL Encoded.  
   - Body Parameters:  
     - Name: `oppId`  
     - Value: Expression `{{$json.id}}` (from input).  
   - Connect `Only New Grants` output to this node.

6. **Add OpenAI Chat Model for Summarization**  
   - Add **Langchain OpenAI Chat Model** node `OpenAI Chat Model`.  
   - Configure with your OpenAI API credentials.  
   - Connect as AI language model to `Summarize Synopsis`.

7. **Summarize Grant Synopsis**  
   - Add **Information Extractor** node `Summarize Synopsis`.  
   - Text field expression:  
     ```
     Agency: {{$json.synopsis.agencyName}}
     Title: {{$json.opportunityTitle}}
     Synopsis: {{$json.synopsis.synopsisDesc}}
     ```  
   - System Prompt: "You've been given a grant opportunity listing. Help summarize the opportunity in simple terms."  
   - Schema: Object with properties `goal` (string|null), `duration` (string), `success_criteria` (array of strings), `good_to_know` (array of strings).  
   - Connect `Get Grant Details` output and set `OpenAI Chat Model` as AI model.

8. **Add OpenAI Chat Model for Eligibility**  
   - Add another **Langchain OpenAI Chat Model** node `OpenAI Chat Model1`.  
   - Configure with OpenAI API credentials.  
   - Connect as AI language model to `Eligibility Factors`.

9. **Determine Eligibility Factors**  
   - Add **Information Extractor** node `Eligibility Factors`.  
   - Text expression:  
     ```
     Agency: {{$json.synopsis.agencyName}}
     Title: {{$json.opportunityTitle}}
     Synopsis: {{$json.synopsis.synopsisDesc}}
     Eligibility: {{$json.synopsis.applicantEligibilityDesc}}
     ```  
   - System prompt contains detailed company description and instructions to assess eligibility (copy from existing node).  
   - Schema: Object with `eligibility_matches` (array of strings).  
   - Connect `Get Grant Details` output and set `OpenAI Chat Model1` as AI model.

10. **Merge AI Outputs**  
    - Add **Merge** node `Merge`.  
    - Mode: Combine by Position.  
    - Connect `Summarize Synopsis` and `Eligibility Factors` outputs to this node.

11. **Save Results to Airtable**  
    - Add **Airtable** node `Save to Tracker`.  
    - Operation: Create.  
    - Connect with your Airtable base and table for grant tracking (use the sample base or your own).  
    - Map fields from the merged output and grant details accordingly (URL, Title, Agency, Eligibility?, etc.).  
    - Connect `Merge` output to this node.  
    - Set Airtable API credentials.

12. **Create Scheduled Trigger for Email Generation**  
    - Add **Schedule Trigger** node `Everyday @ 9am`.  
    - Set to run daily at 9:00 AM.

13. **Fetch New Eligible Grants from Airtable**  
    - Add **Airtable** node `Get New Eligible Grants Today`.  
    - Operation: Search.  
    - Filter Formula:  
      ```
      AND(
        {Status} = 'New',
        {Eligibility?} = 'Yes',
        IS_SAME(DATETIME_FORMAT(Created, 'YYYY-MM-DD'), DATETIME_FORMAT(TODAY(), 'YYYY-MM-DD'))
      )
      ```  
    - Connect `Everyday @ 9am` output to this node.

14. **Generate Email HTML Template**  
    - Add **HTML** node `Generate Email`.  
    - Paste the provided full HTML email template content.  
    - Use expressions to loop over input grants and populate fields (Title, Agency, URL, Synopsis, Success Criteria, Dates, etc.).  
    - Set to execute once.  
    - Connect `Get New Eligible Grants Today` output to this node.

15. **Retrieve Subscribers from Airtable**  
    - Add **Airtable** node `Get Subscribers`.  
    - Operation: Search.  
    - Filter Formula: `{Status} = 'Active'`.  
    - Connect `Generate Email` output to this node.

16. **Send Email to Subscribers via Gmail**  
    - Add **Gmail** node `Send Subscriber Email`.  
    - Set Recipient To: Expression `{{$json.Email}}` (from subscriber records).  
    - Subject: "Daily Newsletter for Interesting US Grants".  
    - Message: Expression `{{$node["Generate Email"].json["html"]}}`.  
    - Connect `Get Subscribers` output to this node.  
    - Configure Gmail OAuth2 credentials.

17. **Connect Nodes Properly**  
    - Connect all nodes following described order.  
    - Ensure AI model nodes are linked as AI language models to their respective Information Extractor nodes.

18. **Add Sticky Notes for Documentation (Optional)**  
    - Add sticky notes as per original workflow to explain blocks and provide helpful links.

---

### 5. General Notes & Resources

| Note Content                                                                                                                               | Context or Link                                                                                   |
|--------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| This template can be adapted to non-AI grants or other lead sources by adjusting the HTTP request endpoint and deduplication key.         | -                                                                                               |
| Sample Airtable base used: [https://airtable.com/appiNoPRvhJxz9crl/shrRdP6zstgsxjDKL](https://airtable.com/appiNoPRvhJxz9crl/shrRdP6zstgsxjDKL) | Airtable database for storing grants and subscribers.                                           |
| Remove Duplicates node works across workflow executions, simplifying duplicate tracking without manual management.                        | [n8n Remove Duplicates docs](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.removeduplicates/) |
| Information Extractor node uses prompts and schemas to parse AI output into structured JSON for easier processing.                        | [n8n Information Extractor docs](https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.information-extractor/) |
| Gmail node supports sending HTML emails and requires OAuth2 credentials.                                                                   | [n8n Gmail node docs](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.gmail/)   |
| Scheduled triggers allow automation at specific times, enabling daily fetch and email routines.                                            | [n8n Scheduled Trigger docs](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.scheduletrigger) |
| Join n8n community for support and tips: [Discord](https://discord.com/invite/XPKeKXeB7d), [Forum](https://community.n8n.io/)               | Community support channels.                                                                      |

---

This document fully describes the workflow structure, node functions, configuration, and how to recreate it in n8n without the original JSON file. It includes considerations for potential errors and extensions.