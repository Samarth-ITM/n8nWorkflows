Discover Decision Makers by Responsibilities (not Titles) with Octave & Airtable

https://n8nworkflows.xyz/workflows/discover-decision-makers-by-responsibilities--not-titles--with-octave---airtable-7620


# Discover Decision Makers by Responsibilities (not Titles) with Octave & Airtable

### 1. Workflow Overview

This workflow automates the discovery of relevant decision-makers within target companies by focusing on actual responsibilities rather than job titles, addressing the common challenge of inconsistent or misleading job titles in prospecting. It is designed primarily for Sales Development Representatives (SDRs), Account-Based Marketing (ABM) professionals, and Revenue Operations teams who require more accurate contact identification to improve outreach effectiveness.

The workflow is logically composed of these blocks:

- **1.1 Trigger Initiation:** Manual or scheduled start of the workflow.
- **1.2 Target Account Retrieval:** Loading target companies from an Airtable base.
- **1.3 Intelligent Contact Discovery:** Using Octave’s AI prospector to find relevant contacts based on company domain and responsibilities.
- **1.4 Contact Data Export:** Writing discovered contacts back into Airtable for further use.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger Initiation

**Overview:**  
This block initiates the workflow manually (or optionally via schedule), ensuring controlled execution when desired.

**Nodes Involved:**  
- Manual Workflow Trigger

**Node Details:**

- **Manual Workflow Trigger**  
  - **Type & Role:** Manual trigger node; user-activated start point.  
  - **Configuration:** Default manual trigger with no parameters, can be replaced with a schedule trigger if needed.  
  - **Expressions/Variables:** None needed here.  
  - **Input/Output:** No inputs; outputs to "Get Target Accounts" node.  
  - **Failure Modes:** N/A.  
  - **Sub-Workflow:** None.  

---

#### 2.2 Target Account Retrieval

**Overview:**  
This block fetches the list of target accounts from Airtable, which serves as the data source for companies to prospect.

**Nodes Involved:**  
- Get Target Accounts

**Node Details:**

- **Get Target Accounts**  
  - **Type & Role:** Airtable node; retrieves records from a specified base and table.  
  - **Configuration:**  
    - Operation: Search (fetch all or filtered records).  
    - Base and Table: Configured with user’s Airtable base ID and accounts table ID (replace placeholders).  
  - **Expressions/Variables:** None directly, but output JSON includes fields such as `Company Domain`.  
  - **Input/Output:** Input from Manual Workflow Trigger; outputs company data to Octave prospector node.  
  - **Failure Modes:**  
    - Authentication errors if Airtable credentials are invalid.  
    - API rate limits or connectivity issues.  
    - Empty or malformed data could cause downstream failures.  
  - **Sub-Workflow:** None.

---

#### 2.3 Intelligent Contact Discovery

**Overview:**  
This block uses Octave’s AI prospector to find contacts responsible for relevant areas within each target company based on the company domain.

**Nodes Involved:**  
- Discover Relevant Contacts

**Node Details:**

- **Discover Relevant Contacts**  
  - **Type & Role:** Octave node; performs AI-driven prospecting using Octave agent.  
  - **Configuration:**  
    - Operation: `runProspector` to perform contact discovery.  
    - Input: Company domain dynamically mapped from `Company Domain` field of each Airtable account record (`={{ $json['Company Domain'] }}`).  
    - Agent ID: User must configure with their specific Octave prospector agent ID.  
  - **Expressions/Variables:** Uses expression to pass the current company domain to Octave.  
  - **Input/Output:** Receives company data from Airtable node; outputs discovered contacts data to Airtable save node.  
  - **Failure Modes:**  
    - Authentication errors if Octave API credentials are invalid.  
    - Timeout or API call failures.  
    - No contacts found for a domain.  
    - Unexpected data structure from Octave.  
  - **Sub-Workflow:** None.

---

#### 2.4 Contact Data Export

**Overview:**  
This block saves the discovered contacts back into Airtable, allowing teams to review and use the enriched contact data.

**Nodes Involved:**  
- Save Discovered Contacts

**Node Details:**

- **Save Discovered Contacts**  
  - **Type & Role:** Airtable node; creates new records in a specified contacts table.  
  - **Configuration:**  
    - Operation: Create.  
    - Base and Table: Configured with user’s Airtable base and contacts table IDs (placeholders to be replaced).  
    - Mapping: Fields mapped explicitly from Octave contact data: First Name, Last Name, Job Title, Company Name, Company Domain, LinkedIn Profile.  
  - **Expressions/Variables:** Maps each contact’s details using expressions like `={{ $json.contacts[0].contact.firstName }}` to handle nested JSON from Octave.  
  - **Input/Output:** Input from Octave prospector; no output nodes.  
  - **Failure Modes:**  
    - Authentication or permission errors with Airtable API.  
    - Data mapping failures if Octave output structure changes.  
    - Duplicate record handling depends on Airtable’s settings (none configured here).  
  - **Sub-Workflow:** None.

---

### 3. Summary Table

| Node Name                 | Node Type              | Functional Role                   | Input Node(s)            | Output Node(s)             | Sticky Note                                                                                         |
|---------------------------|------------------------|---------------------------------|--------------------------|----------------------------|---------------------------------------------------------------------------------------------------|
| Manual Workflow Trigger   | Manual Trigger         | Workflow start point             | —                        | Get Target Accounts         | 🚀 START HERE Manual trigger to start. Can change to schedule.                                    |
| Get Target Accounts       | Airtable               | Retrieve target account records  | Manual Workflow Trigger  | Discover Relevant Contacts  | 📋 ACCOUNT LIST Replace with your data source. Configure account table.                           |
| Discover Relevant Contacts| Octave AI Prospector   | Find relevant contacts by domain| Get Target Accounts       | Save Discovered Contacts    | 🔍 INTELLIGENT PROSPECTOR Context-aware contact discovery. Replace agent ID & configure.         |
| Save Discovered Contacts  | Airtable               | Save contacts to Airtable        | Discover Relevant Contacts| —                          | 💾 CONTACT OUTPUT Save discovered contacts. Configure output table.                              |
| Sticky Note - Main Overview| Sticky Note           | Overview and summary             | —                        | —                          | 🎯 INTELLIGENT CONTACT PROSPECTING FOR: SDR teams, ABM professionals, RevOps who need to find the right people based on actual responsibilities, not just job titles. SOLVES: Traditional prospecting relies on job title matching but titles vary wildly. You miss the "Head of Platform" who owns your use case while searching for "VP of Engineering". WORKS: 1. Manual trigger starts workflow 2. Read target accounts from Airtable 3. Octave prospector finds relevant contacts 4. Contacts exported back to Airtable SETUP: Airtable credentials + account list, Octave prospector agent, contact output table CUSTOMIZE: Configure prospector personas, responsibilities, org levels. Adapt data sources (CRM, spreadsheets). Adjust contact selection criteria and output fields. |
| Sticky Note - Trigger Setup| Sticky Note           | Trigger explanation              | —                        | —                          | 🚀 START HERE Manual trigger to start. Can change to schedule.                                    |
| Sticky Note - Account Source| Sticky Note          | Source data explanation          | —                        | —                          | 📋 ACCOUNT LIST Replace with your data source. Configure account table.                           |
| Sticky Note - Prospector Agent| Sticky Note         | Prospector configuration         | —                        | —                          | 🔍 INTELLIGENT PROSPECTOR Context-aware contact discovery. Replace agent ID & configure.         |
| Sticky Note - Contact Output| Sticky Note           | Output data explanation          | —                        | —                          | 💾 CONTACT OUTPUT Save discovered contacts. Configure output table.                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Add a "Manual Trigger" node as the workflow entry point. No configuration needed unless scheduling is desired.

2. **Create Airtable Node to Get Target Accounts**  
   - Add an "Airtable" node.  
   - Set operation to "Search".  
   - Select your Airtable credentials.  
   - Configure Base to your target accounts base.  
   - Select the table that contains your account list.  
   - Connect the Manual Trigger node output to this node’s input.

3. **Create Octave Node for Contact Discovery**  
   - Add an "Octave" node (OctaveHQ integration).  
   - Set operation to "runProspector".  
   - Configure Octave API credentials.  
   - Set "AgentOId" to your Octave prospector agent ID.  
   - In the "companyDomain" field, use the expression: `={{ $json['Company Domain'] }}` to dynamically pass the domain from Airtable data.  
   - Connect the "Get Target Accounts" node output to this node.

4. **Create Airtable Node to Save Discovered Contacts**  
   - Add another "Airtable" node.  
   - Set operation to "Create".  
   - Use your Airtable credentials.  
   - Set Base and Table to your contacts list base/table.  
   - Map the contact fields from the Octave output JSON:  
     - Job Title: `={{ $json.contacts[0].contact.title }}`  
     - First Name: `={{ $json.contacts[0].contact.firstName }}`  
     - Last Name: `={{ $json.contacts[0].contact.lastName }}`  
     - Company Name: `={{ $json.contacts[0].contact.companyName }}`  
     - Company Domain: `={{ $json.contacts[0].contact.companyDomain }}`  
     - LinkedIn Profile: `={{ $json.contacts[0].contact.profileUrl }}`  
   - Connect the Octave node output to this node.

5. **Test Workflow**  
   - Save and execute the workflow manually to verify data flows correctly.  
   - Ensure Airtable credentials have write permissions.  
   - Confirm Octave API credentials and agent ID are valid.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                  | Context or Link                                               |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------|
| The workflow addresses a common challenge in B2B prospecting by focusing on responsibilities rather than job titles, which vary widely and often mislead traditional filters.                                                                   | Main workflow goal as described in the main sticky note.      |
| Replace all placeholder Airtable base IDs, table IDs, and Octave agent IDs with your actual account-specific values before running.                                                                                                         | Configuration instructions in sticky notes and node settings. |
| Octave Prospector requires a valid API key and an agent configured with targeted personas and responsibilities aligned to your use case.                                                                                                    | Octave node configuration details.                            |
| Airtable rate limits and API quotas might impact large batch processing; consider pagination or batch sizes if expanding.                                                                                                                    | General Airtable API consideration.                           |
| For scheduled runs, replace the Manual Trigger node with a Cron or Schedule Trigger node available in n8n.                                                                                                                                   | Trigger customization note from sticky note.                  |
| Workflow is designed to be adapted to use other data sources such as CRM exports or spreadsheets by replacing the Airtable input node accordingly.                                                                                            | Customization note from main sticky note.                      |
| Useful resource: [Octave AI Prospecting](https://octave.ai) and [Airtable API Documentation](https://airtable.com/api) for further customization and troubleshooting.                                                                        | External links for API references.                            |

---

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.