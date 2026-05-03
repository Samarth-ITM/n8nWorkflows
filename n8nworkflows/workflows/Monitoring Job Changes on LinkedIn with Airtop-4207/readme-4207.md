Monitoring Job Changes on LinkedIn with Airtop

https://n8nworkflows.xyz/workflows/monitoring-job-changes-on-linkedin-with-airtop-4207


# Monitoring Job Changes on LinkedIn with Airtop

### 1. Workflow Overview

This n8n workflow, titled **"Monitoring Job Changes on LinkedIn with Airtop"**, is designed to monitor and extract job change information from a LinkedIn user's network feed. It targets professionals who want to track career movements of their LinkedIn connections for lead enrichment, CRM updates, or timely outreach.

The workflow consists of three main logical blocks:

- **1.1 Input Reception:** Manual trigger to initiate the extraction process.
- **1.2 Data Extraction and Classification:** Uses Airtop to scrape LinkedIn’s "Job Changes" feed, extracting key details and categorizing new positions.
- **1.3 Data Formatting:** Structures and prepares the extracted data for further processing or integration.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:** This block starts the workflow manually, allowing the user to control when to run the job change extraction.
- **Nodes Involved:** `When clicking ‘Test workflow’`
- **Node Details:**

  - **Name:** When clicking ‘Test workflow’  
  - **Type:** Manual Trigger  
  - **Technical Role:** Entry point to manually start the workflow on demand.  
  - **Configuration:** Default manual trigger with no parameters.  
  - **Expressions/Variables:** None.  
  - **Input/Output Connections:** No input; output connects to `Extract Job Changes`.  
  - **Version Requirements:** Compatible with n8n version supporting manual triggers (standard node).  
  - **Potential Failures:** None expected; manual trigger is user-initiated.  
  - **Sub-workflow:** None.

#### 1.2 Data Extraction and Classification

- **Overview:** This block scrapes LinkedIn’s "Job Changes" feed using Airtop, extracting details about recent job changes and classifying new positions by function.  
- **Nodes Involved:** `Extract Job Changes`  
- **Node Details:**

  - **Name:** Extract Job Changes  
  - **Type:** Airtop Node (Custom integration for browser automation and scraping)  
  - **Technical Role:** Performs authenticated web scraping and AI-powered extraction from LinkedIn job changes page.  
  - **Configuration Choices:**
    - URL set to LinkedIn job changes feed: `https://www.linkedin.com/mynetwork/catch-up/job_changes/`
    - Prompt instructs extraction of 5 job changes with fields: name, new position, LinkedIn profile URL, and classification into functions like marketing, sales, HR, executive.
    - Output schema explicitly defines expected JSON structure with strong typing and required fields.
    - Uses Airtop profile named `AmirLinkedin` with `new` session mode ensuring a fresh browser session.
  - **Expressions/Variables:** Prompt text and JSON schema hardcoded in parameters for precise extraction.
  - **Input/Output Connections:** Input from manual trigger node; output passes extracted JSON to `Edit Fields`.
  - **Version Requirements:** Requires Airtop API credentials configured in n8n; supports schema validation.
  - **Potential Failures:**
    - Authentication errors if Airtop or LinkedIn credentials are invalid or expired.
    - Timeout or navigation failure if LinkedIn page structure changes or network issues occur.
    - Schema mismatch if LinkedIn feed format changes or prompt fails to parse data correctly.
  - **Sub-workflow:** None.

#### 1.3 Data Formatting

- **Overview:** This block cleans and restructures the data returned by Airtop to a simplified format for downstream consumption or integration.
- **Nodes Involved:** `Edit Fields`
- **Node Details:**

  - **Name:** Edit Fields  
  - **Type:** Set Node (Data manipulation)  
  - **Technical Role:** Copies the nested extracted response (`data.modelResponse`) into a simplified JSON object to isolate relevant data.  
  - **Configuration Choices:** Uses expression to assign `data.modelResponse` from Airtop node output to the current node’s JSON directly.  
  - **Expressions/Variables:** `={{ $json.data.modelResponse }}` to extract payload.  
  - **Input/Output Connections:** Input from `Extract Job Changes`; output not connected further but ready for downstream use.  
  - **Version Requirements:** Compatible with n8n version supporting Set node version 3.4+.  
  - **Potential Failures:** Expression errors if prior node output changes format or is empty.  
  - **Sub-workflow:** None.

---

### 3. Summary Table

| Node Name               | Node Type        | Functional Role                   | Input Node(s)               | Output Node(s)           | Sticky Note                                                                                      |
|-------------------------|------------------|---------------------------------|-----------------------------|--------------------------|-------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger   | Manual start of workflow          | —                           | Extract Job Changes       | README # Monitoring Job Changes on LinkedIn, Use case, setup instructions, and next steps info. |
| Extract Job Changes      | Airtop           | Scrape and extract job changes   | When clicking ‘Test workflow’ | Edit Fields              | README # Monitoring Job Changes on LinkedIn, Use case, setup instructions, and next steps info. |
| Edit Fields             | Set              | Format and simplify extracted data | Extract Job Changes          | —                        | README # Monitoring Job Changes on LinkedIn, Use case, setup instructions, and next steps info. |
| Sticky Note             | Sticky Note      | Documentation and usage guidance | —                           | —                        | README # Monitoring Job Changes on LinkedIn, Use case, setup instructions, and next steps info. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**
   - Node Type: Manual Trigger
   - Name: `When clicking ‘Test workflow’`
   - No parameters needed.
   - Position: Start of workflow.

2. **Create Airtop Node for Extraction**
   - Node Type: Airtop
   - Name: `Extract Job Changes`
   - Parameters:
     - URL: `https://www.linkedin.com/mynetwork/catch-up/job_changes/`
     - Resource: `extraction`
     - Operation: `query`
     - Prompt:  
       ```
       This is a list of job changes. Extract 5 job changes. For each job change, extract the name of the person, the new position, and the person's LinkedIn profile URL of the person.

       Also, classify every new position to a function like marketing, sales, HR, executive, and so on.
       ```
     - Profile Name: `AmirLinkedin` (must be preconfigured in Airtop Portal with LinkedIn login)
     - Session Mode: `new` (fresh browser session)
     - Additional Fields: Output JSON schema defining the structure and required fields:
       - `job_changes` array of objects with `name`, `new_position`, `linkedin_profile_url`, `position_function`
   - Credentials: Airtop API credentials configured in n8n.
   - Position: Connect output of Manual Trigger node to this node.

3. **Create Set Node for Formatting**
   - Node Type: Set
   - Name: `Edit Fields`
   - Parameters:
     - Assign `data.modelResponse` from previous Airtop node output using expression: `={{ $json.data.modelResponse }}`
   - Position: Connect output of Airtop node to this node.

4. **Add Sticky Note for Documentation**
   - Node Type: Sticky Note
   - Name: `Sticky Note`
   - Content:  
     ```
     README
     # Monitoring Job Changes on LinkedIn
     ## Use Case
     This automation tracks job changes among your LinkedIn connections and extracts relevant details. It's ideal for triggering timely outreach, updating CRM records, or feeding lead scoring workflows based on new roles.

     ## What This Automation Does
     It scrapes your LinkedIn "Job Changes" feed and returns:
     - Name of the person
     - Their new position
     - LinkedIn profile URL
     - Functional category (e.g., marketing, sales, HR, executive)

     Each run processes 5 job changes at a time.

     ## How It Works
     1. Manual Trigger
     2. Airtop Enrichment
     3. Formatting

     ## Setup Requirements
     1. [Airtop Profile](https://portal.airtop.ai/browser-profiles) connected to LinkedIn
     2. Airtop API key configured in n8n
     3. A LinkedIn account with a populated “Job Changes” feed

     ## Next Steps
     - Automate Alerts (Slack, email, CRM)
     - Enrich and Score Leads
     - Customize scope to extract more or filtered job changes
     ```
   - Position: Place near the trigger node or workflow start for visibility.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                      | Context or Link                                  |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------|
| Airtop Profile must be linked to a LinkedIn account with access to the "Job Changes" feed for the scraping to work correctly.                                                                                                    | https://portal.airtop.ai/browser-profiles       |
| Airtop API credentials must be configured in n8n for the `Extract Job Changes` node to authenticate and operate.                                                                                                                | n8n Credentials setup                           |
| The workflow processes 5 job changes per run but can be customized to extract more or apply filters by modifying the Airtop prompt or output schema.                                                                             | Workflow customization                           |
| Consider integrating downstream nodes for Slack, Email, or CRM to automate notifications or lead enrichment based on the extracted data.                                                                                        | Next steps in README note                         |
| The JSON schema in the Airtop node enforces strict data structure; LinkedIn UI changes or network issues may cause extraction or schema validation failures. Monitor execution logs for errors and update prompt/schema if needed. | Stability considerations                          |

---

**Disclaimer:** The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.