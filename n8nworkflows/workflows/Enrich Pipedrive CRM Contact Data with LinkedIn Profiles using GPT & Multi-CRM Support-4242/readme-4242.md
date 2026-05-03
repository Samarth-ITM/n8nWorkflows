Enrich Pipedrive CRM Contact Data with LinkedIn Profiles using GPT & Multi-CRM Support

https://n8nworkflows.xyz/workflows/enrich-pipedrive-crm-contact-data-with-linkedin-profiles-using-gpt---multi-crm-support-4242


# Enrich Pipedrive CRM Contact Data with LinkedIn Profiles using GPT & Multi-CRM Support

### 1. Workflow Overview

This workflow titled **"Enrich Pipedrive CRM Contact Data with LinkedIn Profiles using GPT & Multi-CRM Support"** automates the enrichment of CRM contact data by integrating LinkedIn professional insights. It targets Pipedrive and HubSpot CRMs and uses LinkedIn data enrichment powered by an AI agent based on OpenAI's GPT-4o model along with HDW LinkedIn API nodes.

**Use Cases:**
- Automatically enrich new contacts or updated contacts flagged for enrichment in Pipedrive or HubSpot.
- Fetch and analyze LinkedIn profiles matching contact emails or, failing that, by searching with name, company, and location.
- Generate professional summaries and recent LinkedIn post analyses.
- Update CRM contacts with enriched LinkedIn profile data for improved sales and marketing insights.

**Logical Blocks:**

- **1.1 Triggers**: Detect new or updated contacts in Pipedrive and HubSpot.
- **1.2 Data Retrieval & Conditional Checks**: Retrieve contact and company data, evaluate enrichment flags.
- **1.3 AI Data Enrichment Agents**: Use GPT-powered agents to process and analyze contacts with LinkedIn data.
- **1.4 LinkedIn Data Collection**: Fetch LinkedIn profiles by email or search, get profile details and posts via HDW LinkedIn nodes.
- **1.5 CRM Data Update**: Write enriched LinkedIn data back into Pipedrive or HubSpot.
- **1.6 Notes & Documentation**: Sticky notes provide setup instructions, field mappings, and general workflow guidance.

---

### 2. Block-by-Block Analysis

#### 1.1 Triggers

**Overview:**  
This block listens for relevant CRM events to start enrichment workflows.

**Nodes Involved:**  
- Pipedrive Trigger New Contact  
- Pipedrive Trigger Update Data for Existing Client  
- HubSpot Trigger  

**Node Details:**  

- **Pipedrive Trigger New Contact**  
  - Type: Trigger node (Pipedrive)  
  - Watches for creation of new "person" entities in Pipedrive CRM.  
  - Starts the enrichment process for new contacts.  
  - Possible failures: webhook misconfiguration, Pipedrive API rate limits.

- **Pipedrive Trigger Update Data for Existing Client**  
  - Type: Trigger node (Pipedrive)  
  - Watches for changes on existing "person" entities.  
  - Used to trigger enrichment only if flagged.  

- **HubSpot Trigger**  
  - Type: Trigger node (HubSpot)  
  - Watches for contact creation and property change events, specifically the "need_enrichment" flag change.  
  - Possible failures: webhook not registered, permission issues.

---

#### 1.2 Data Retrieval & Conditional Checks

**Overview:**  
Retrieves contact data from HubSpot and company data from Pipedrive, and checks if enrichment is needed.

**Nodes Involved:**  
- HubSpot Get Contact Data  
- If1 (checks event type)  
- If2 (checks enrichment flag)  
- Enrich Flag is True (checks Pipedrive enrichment flag)  
- Get Company from CRM  
- Get Company from CRM1  

**Node Details:**  

- **HubSpot Get Contact Data**  
  - Type: HubSpot API node  
  - Fetches contact details like firstname, lastname, email, company using contactId from trigger.  
  - Uses "appToken" authentication.  
  - Failure cases: invalid contactId, token expiration.

- **If1**  
  - Type: If node (version 2.2)  
  - Checks if HubSpot event is contact creation. Routes accordingly.  
  - Conditions use strict, case-sensitive string comparison.

- **If2**  
  - Type: If node (version 2.2)  
  - Checks if HubSpot event is property change AND the "need_enrichment" property is true.  
  - Ensures enrichment runs only when requested.

- **Enrich Flag is True**  
  - Type: If node  
  - Checks a specific Pipedrive custom field ID equals 29 (likely "need enrichment" flag).  
  - Routes only flagged contacts to enrichment.

- **Get Company from CRM** and **Get Company from CRM1**  
  - Type: Pipedrive API nodes  
  - Retrieves organization info by organizationId (dynamic from AI variables).  
  - Used for fallback search parameters when LinkedIn profile not found by email.  
  - Failure: invalid org id, API error.

---

#### 1.3 AI Data Enrichment Agents

**Overview:**  
Central AI agents process CRM contact data, orchestrate LinkedIn profile search, retrieval, analysis, and prepare CRM update data.

**Nodes Involved:**  
- Data Enrichment AI Agent  
- Data Enrichment AI Agent1  
- Data Enrichment AI Agent2  
- OpenAI Chat Model  
- OpenAI Chat Model1  
- OpenAI Chat Model2  

**Node Details:**  

- **Data Enrichment AI Agent / Data Enrichment AI Agent1 / Data Enrichment AI Agent2**  
  - Type: LangChain AI Agent nodes using OpenAI GPT-4o model  
  - Receive structured user data (email, name, org id/company) from previous nodes.  
  - System prompt defines steps: email-based LinkedIn profile search → fallback search → profile details → post retrieval → summarization → CRM update.  
  - Use custom tools like `get_linkedin_email_user`, `get_organisation_info`, `get_linkedin_profile`, `get_linkedin_user_posts`, `update_pipedrive_data`, and `search_linkedin_users`.  
  - Designed to handle Pipedrive and HubSpot variants with minor prompt differences.  
  - Edge cases: API timeouts, no LinkedIn profile found, malformed input, rate limits.

- **OpenAI Chat Model / OpenAI Chat Model1 / OpenAI Chat Model2**  
  - Type: LangChain OpenAI GPT-4o model nodes  
  - Serve as underlying language models for AI Agents.  
  - Credentials configured with OpenAI API key.  
  - No special parameters beyond model selection.  
  - Possible failures: API key invalid, quota exceeded, network failures.

---

#### 1.4 LinkedIn Data Collection

**Overview:**  
This block uses HDW LinkedIn nodes to search and retrieve LinkedIn profiles and posts based on email or search criteria.

**Nodes Involved:**  
- HDW Get LinkedIn profile by Email  
- HDW Get LinkedIn profile by Email1  
- HDW Get LinkedIn profile by Email2  
- HDW Search LinkedIn Profile  
- HDW Search LinkedIn Profile1  
- HDW Search LinkedIn Profile2  
- HDW Get LinkedIn Profile Details  
- HDW Get LinkedIn Profile Details1  
- HDW Get LinkedIn Profile Details2  
- HDW Get LinkedIn Profile Posts  
- HDW Get LinkedIn Profile Posts1  
- HDW Get LinkedIn Profile Posts2  

**Node Details:**  

- **HDW Get LinkedIn profile by Email / Email1 / Email2**  
  - Type: HDW LinkedIn community node  
  - Searches LinkedIn profiles by email (count=1).  
  - Uses dynamic email input extracted from AI context variables.  
  - Failures: no profile found, API key limits, invalid email input.

- **HDW Search LinkedIn Profile / Profile1 / Profile2**  
  - Type: HDW LinkedIn community node  
  - Searches LinkedIn profiles by keywords, location, first name, last name, company keywords.  
  - Used as alternative when email search fails.  
  - Configured with dynamic AI variables for flexible search.  
  - Failures: no matches, rate limiting.

- **HDW Get LinkedIn Profile Details / Details1 / Details2**  
  - Type: HDW LinkedIn node  
  - Retrieves detailed profile info for a LinkedIn user (by alias).  
  - Input driven from AI variables.  
  - Failures: invalid user alias, API error.

- **HDW Get LinkedIn Profile Posts / Posts1 / Posts2**  
  - Type: HDW LinkedIn node  
  - Retrieves recent posts given a LinkedIn user URN.  
  - Used to analyze professional activity and interests.  
  - Failures: no posts available, API limits.

---

#### 1.5 CRM Data Update

**Overview:**  
Updates CRM contacts (Pipedrive or HubSpot) with enriched LinkedIn data: profile URL, profile summary, and post summaries.

**Nodes Involved:**  
- Update data in Pipedrive  
- Update data in Pipedrive1  
- Update data in HubSpot  

**Node Details:**  

- **Update data in Pipedrive / Update data in Pipedrive1**  
  - Type: Pipedrive API nodes  
  - Update person records with custom properties for LinkedIn data fields.  
  - Uses dynamic AI-generated values for post summary, profile URL, and profile summary.  
  - Requires correct mapping of custom field IDs.  
  - Failure points: invalid personId, missing custom fields, API quota.

- **Update data in HubSpot**  
  - Type: HubSpot API node  
  - Updates contact properties with LinkedIn post summary, profile summary, and LinkedIn URL.  
  - Uses app token authentication.  
  - Failure points: invalid email, missing properties, token expiry.

---

#### 1.6 Notes & Documentation

**Overview:**  
Sticky Notes provide detailed instructions, setup guidance, and reminders about custom fields and field mappings.

**Nodes Involved:**  
- Sticky Note (multiple) including Sticky Note11 with workflow documentation  

**Node Details:**  

- Provide instructions on:  
  - Creating custom fields in Pipedrive and HubSpot  
  - Checking field mapping in update nodes  
  - Overall workflow purpose and setup  
  - Links to HDW LinkedIn node installation and CRM field configuration guides  
- Essential for administrators setting up or modifying the workflow.

---

### 3. Summary Table

| Node Name                          | Node Type                                    | Functional Role                        | Input Node(s)                             | Output Node(s)                            | Sticky Note                                         |
|-----------------------------------|----------------------------------------------|-------------------------------------|------------------------------------------|------------------------------------------|-----------------------------------------------------|
| Pipedrive Trigger New Contact      | Pipedrive Trigger                            | Trigger on new Pipedrive contacts   |                                          | Data Enrichment AI Agent                  |                                                     |
| Data Enrichment AI Agent           | LangChain AI Agent                           | AI processing & orchestration       | Pipedrive Trigger New Contact, OpenAI Chat Model, HDW nodes | Update data in Pipedrive, HDW LinkedIn nodes |                                                     |
| OpenAI Chat Model                 | LangChain OpenAI Model                       | Underlying GPT-4o model             |                                          | Data Enrichment AI Agent                  |                                                     |
| HDW Get LinkedIn profile by Email | HDW LinkedIn API Node                        | Search LinkedIn by email            | Data Enrichment AI Agent                  | HDW Search LinkedIn Profile, HDW Get LinkedIn Profile Details |                                                     |
| HDW Search LinkedIn Profile        | HDW LinkedIn API Node                        | Search LinkedIn by keywords         | Data Enrichment AI Agent                  | HDW Get LinkedIn Profile Details          |                                                     |
| HDW Get LinkedIn Profile Details   | HDW LinkedIn API Node                        | Get LinkedIn profile details        | HDW Search LinkedIn Profile, HDW Get LinkedIn profile by Email | HDW Get LinkedIn Profile Posts            |                                                     |
| HDW Get LinkedIn Profile Posts     | HDW LinkedIn API Node                        | Get recent LinkedIn posts           | HDW Get LinkedIn Profile Details          | Data Enrichment AI Agent                  |                                                     |
| Get Company from CRM               | Pipedrive API Node                          | Retrieve company info from CRM      |                                          | Data Enrichment AI Agent                  |                                                     |
| Update data in Pipedrive           | Pipedrive API Node                          | Update Pipedrive contact with LinkedIn data | Data Enrichment AI Agent                  |                                          | Check the fields mapping in the node                 |
| Pipedrive Trigger Update Data for Existing Client | Pipedrive Trigger                    | Trigger on contact update           |                                          | Enrich Flag is True                       |                                                     |
| Enrich Flag is True                | If Node                                    | Check enrichment flag in Pipedrive | Pipedrive Trigger Update Data for Existing Client | Data Enrichment AI Agent2                 |                                                     |
| Data Enrichment AI Agent2          | LangChain AI Agent                           | AI processing for updates           | Enrich Flag is True, OpenAI Chat Model1, HDW linked nodes | Update data in Pipedrive1                 | Check the fields mapping in the node                 |
| Update data in Pipedrive1          | Pipedrive API Node                          | Update existing Pipedrive contacts  | Data Enrichment AI Agent2                 |                                          | Check the fields mapping in the node                 |
| HubSpot Trigger                   | HubSpot Trigger                             | Trigger on HubSpot contact events   |                                          | HubSpot Get Contact Data                  |                                                     |
| HubSpot Get Contact Data           | HubSpot API Node                            | Retrieve HubSpot contact data       | HubSpot Trigger                          | If1                                     |                                                     |
| If1                              | If Node                                    | Check if event is contact creation  | HubSpot Get Contact Data                  | Data Enrichment AI Agent1, If2            |                                                     |
| If2                              | If Node                                    | Check if enrichment flag is true    | If1                                      | Data Enrichment AI Agent1                 |                                                     |
| Data Enrichment AI Agent1          | LangChain AI Agent                           | AI processing for HubSpot contacts  | If2, OpenAI Chat Model2, HDW linked nodes | Update data in HubSpot                     |                                                     |
| Update data in HubSpot             | HubSpot API Node                            | Update HubSpot contact with LinkedIn data | Data Enrichment AI Agent1                 |                                          | Check the fields mapping in the node                 |
| HDW Get LinkedIn profile by Email1 | HDW LinkedIn API Node                      | Email-based LinkedIn profile search | Data Enrichment AI Agent2                 | HDW Search LinkedIn Profile1, HDW Get LinkedIn Profile Details1 |                                                     |
| HDW Search LinkedIn Profile1       | HDW LinkedIn API Node                      | Keyword-based LinkedIn profile search | HDW Get LinkedIn profile by Email1       | HDW Get LinkedIn Profile Details1         |                                                     |
| HDW Get LinkedIn Profile Details1  | HDW LinkedIn API Node                      | Retrieve LinkedIn profile details   | HDW Search LinkedIn Profile1              | HDW Get LinkedIn Profile Posts1            |                                                     |
| HDW Get LinkedIn Profile Posts1    | HDW LinkedIn API Node                      | Get LinkedIn posts                  | HDW Get LinkedIn Profile Details1         | Data Enrichment AI Agent2                  |                                                     |
| HDW Get LinkedIn profile by Email2 | HDW LinkedIn API Node                      | Email LinkedIn search (HubSpot)     | Data Enrichment AI Agent1                  | HDW Search LinkedIn Profile2, HDW Get LinkedIn Profile Details2 |                                                     |
| HDW Search LinkedIn Profile2       | HDW LinkedIn API Node                      | Keyword LinkedIn search (HubSpot)   | HDW Get LinkedIn profile by Email2        | HDW Get LinkedIn Profile Details2          |                                                     |
| HDW Get LinkedIn Profile Details2  | HDW LinkedIn API Node                      | Retrieve LinkedIn profile details   | HDW Search LinkedIn Profile2               | HDW Get LinkedIn Profile Posts2             |                                                     |
| HDW Get LinkedIn Profile Posts2    | HDW LinkedIn API Node                      | Get LinkedIn posts                  | HDW Get LinkedIn Profile Details2          | Data Enrichment AI Agent1                   |                                                     |
| Get Company from CRM1              | Pipedrive API Node                          | Retrieve company data (HubSpot flow) | Data Enrichment AI Agent2                 |                                          |                                                     |
| Sticky Note (multiple)             | Sticky Note                                | Documentation and setup instructions |                                          |                                          | See detailed notes in section 5                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Triggers:**

   - Add a **Pipedrive Trigger** node:
     - Action: Create
     - Entity: Person
     - Credentials: Pipedrive API
     - Position accordingly.

   - Add a **Pipedrive Trigger** node:
     - Action: Change
     - Entity: Person
     - Credentials: Pipedrive API

   - Add a **HubSpot Trigger** node:
     - Events: contact.creation, contact.propertyChange (property = need_enrichment)
     - Credentials: HubSpot Developer API

2. **Retrieve Contact Data:**

   - For HubSpot contacts:
     - Add **HubSpot Get Contact Data** node:
       - Operation: Get contact
       - ContactId: From HubSpot Trigger output
       - Properties: firstname, lastname, email, company
       - Auth: appToken

3. **Add Conditional Nodes to Check Events and Flags:**

   - Add If node (If1):
     - Condition: subscriptionType equals "contact.creation" (case-sensitive)
     - Connect HubSpot Get Contact Data → If1

   - Add If node (If2):
     - Condition 1: subscriptionType equals "contact.propertyChange"
     - Condition 2: need_enrichment property equals true
     - Use AND combinator
     - Connect If1 false path → If2

   - Add If node (Enrich Flag is True) for Pipedrive update trigger:
     - Check custom field (ID: 29) equals 29 (indicates enrichment flag)

4. **Add AI Agent Nodes:**

   - Add **LangChain AI Agent** nodes (Data Enrichment AI Agent, Agent1, Agent2):
     - Use OpenAI GPT-4o model via LangChain OpenAI node.
     - Configure prompt with system message describing LinkedIn enrichment process, inputs (email, name, org_id/company), tools available, and output requirements.
     - Connect triggers and conditional nodes to respective AI agents.

   - Add **LangChain OpenAI Chat Model** nodes for GPT-4o model:
     - Connect AI agents to these chat model nodes as language model.

5. **Add HDW LinkedIn Nodes for Data Collection:**

   - Add **HDW Get LinkedIn profile by Email** nodes:
     - Count: 1
     - Email: Dynamic from AI agent input variables.
     - Resource: Email

   - Add **HDW Search LinkedIn Profile** nodes:
     - Keywords, location, first name, last name, company keywords from AI variables.

   - Add **HDW Get LinkedIn Profile Details** nodes:
     - User alias dynamic from AI variables.

   - Add **HDW Get LinkedIn Profile Posts** nodes:
     - URN dynamic from AI variables.

   - Connect these nodes in sequence: Email search → Fallback search → Profile details → Posts.

6. **Retrieve Organization Info from Pipedrive:**

   - Add **Pipedrive Get Organization** nodes:
     - OrganizationId dynamic from AI variables.
     - Connect as fallback for enrichment AI agents.

7. **Add CRM Update Nodes:**

   - Add **Pipedrive Update Person** nodes:
     - PersonId from trigger data.
     - Update custom fields: LinkedIn profile URL, profile summary, LinkedIn posts summary.
     - Map fields carefully with correct field IDs.

   - Add **HubSpot Update Contact** node:
     - Email from HubSpot contact data.
     - Update properties linkedin_url, profile_summary, linkedin_posts_summary.
     - Use appToken authentication.

8. **Connect Nodes:**

   - Triggers → Contact Data Retrieval → Conditionals → AI Agents → HDW LinkedIn nodes → AI Agents → CRM Update nodes.

9. **Add Sticky Notes:**

   - Add multiple sticky notes to describe:
     - Workflow purpose
     - Custom field creation instructions for Pipedrive and HubSpot
     - Field mapping reminders
     - Setup instructions for HDW LinkedIn node and OpenAI credentials

10. **Configure Credentials:**

    - OpenAI API credential with GPT-4o access.
    - Pipedrive API credential.
    - HubSpot Developer API and App Token credentials.
    - HDW LinkedIn API credential from https://app.horizondatawave.ai.

11. **Test Workflow:**

    - Validate triggering on new and updated contacts.
    - Verify LinkedIn profile fetch and enrichment.
    - Confirm CRM updates populate custom fields correctly.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                  | Context or Link                                                                                               |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------|
| Self-hosted n8n instance with HDW LinkedIn community node installed is required. Installation guide: `npm install n8n-nodes-hdw` [https://www.npmjs.com/package/n8n-nodes-hdw](https://www.npmjs.com/package/n8n-nodes-hdw)                         | HDW LinkedIn Node Installation                                                                                 |
| Create custom fields in Pipedrive: LinkedIn Profile, Profile Summary, LinkedIn Posts Summary, Need Enrichment; detailed guide [https://support.pipedrive.com/en/article/custom-fields](https://support.pipedrive.com/en/article/custom-fields)   | Pipedrive CRM Custom Fields Setup                                                                               |
| Create properties in HubSpot: linkedin_url (single-line), profile_summary (multi-line), linkedin_posts_summary (multi-line), need_enrichment (checkbox); guide [https://knowledge.hubspot.com/properties/create-and-edit-properties](https://knowledge.hubspot.com/properties/create-and-edit-properties) | HubSpot CRM Property Setup                                                                                       |
| The AI agents use a structured processing algorithm with graceful error handling to respect privacy and data protection standards.                                                                                                           | Workflow Design                                                                                                 |
| Rate limits in LinkedIn API (via HDW) may require batch processing or delays.                                                                                                                                                                | API Rate Limiting                                                                                                |

---

**Disclaimer:**  
The content above is derived exclusively from an automated n8n workflow. It complies strictly with content policies and contains no illegal, offensive, or protected elements. All data processed is legal and public.