Transf meeting booking into Notion's Task with verified information

https://n8nworkflows.xyz/workflows/transf-meeting-booking-into-notion-s-task-with-verified-information-1325


# Transf meeting booking into Notion's Task with verified information

### 1. Workflow Overview

This workflow automates the process of converting a newly scheduled meeting from Calendly into a verified and enriched task entry in Notion. Its primary use case is to centralize meeting activities and contact information within a Notion database, ensuring that the contact details are accurate and complete by leveraging Dropcontact's enrichment service.

**Logical Blocks:**

- **1.1 Input Reception:** Trigger the workflow on a new Calendly event creation.
- **1.2 Contact Verification & Enrichment:** Use Dropcontact to verify and enrich the contact’s email and related information.
- **1.3 Task Creation in Notion:** Create a new task in a Notion database with the enriched contact and event details.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block starts the workflow by listening for new meeting bookings created in Calendly.

- **Nodes Involved:**  
  - Calendly Trigger

- **Node Details:**

  - **Node Name:** Calendly Trigger  
    - **Type:** `calendlyTrigger` (Trigger node)  
    - **Technical Role:** Listens for a new Calendly event of type `invitee.created` to start the workflow.  
    - **Configuration:**  
      - Event subscribed: `invitee.created` (triggers when an invitee schedules a meeting)  
      - Webhook ID is empty, so it uses automatic webhook creation.  
    - **Input/Output:**  
      - No input (trigger node)  
      - Output connects to Dropcontact node  
    - **Version Requirements:** n8n version supporting Calendly trigger node  
    - **Potential Failures:**  
      - Webhook registration failure due to Calendly API limits or credentials  
      - Missing or revoked Calendly credentials  
      - Network or webhook delivery issues  

#### 1.2 Contact Verification & Enrichment

- **Overview:**  
  This block enriches and verifies the contact’s email and personal details received from Calendly using Dropcontact’s API.

- **Nodes Involved:**  
  - Dropcontact

- **Node Details:**

  - **Node Name:** Dropcontact  
    - **Type:** `dropcontact` (Action node)  
    - **Technical Role:** Enriches the contact email and related data for accuracy and completeness.  
    - **Configuration:**  
      - Email input: Extracted from the Calendly event payload (`{{$json["payload"]["invitee"]["email"]}}`)  
      - Options enabled: `siren` (company identifier), language set to French (`fr`)  
      - Additional fields passed: full name, last name, first name extracted from Calendly payload  
    - **Key Expressions:**  
      - Email: `={{$json["payload"]["invitee"]["email"]}}`  
      - Full name: `={{$json["payload"]["invitee"]["name"]}}`  
      - Last name & First name similarly extracted  
    - **Input/Output:**  
      - Input from Calendly Trigger  
      - Output to Notion node  
    - **Version Requirements:** n8n version supporting Dropcontact node  
    - **Potential Failures:**  
      - API limit exceeded or invalid Dropcontact credentials  
      - Input email missing or malformed  
      - Network latency/timeouts  
      - Partial enrichment results or missing fields  
      
#### 1.3 Task Creation in Notion

- **Overview:**  
  This block creates a new entry (task) in a Notion database with enriched and verified information from Dropcontact and the original Calendly event.

- **Nodes Involved:**  
  - Notion

- **Node Details:**

  - **Node Name:** Notion  
    - **Type:** `notion` (Action node)  
    - **Technical Role:** Inserts a new database page in Notion representing the meeting task.  
    - **Configuration:**  
      - Resource: `databasePage` (creates a new page inside a Notion database)  
      - Database ID: (to be set, required for operation)  
      - Properties mapped:  
        - Date (start & end) from Calendly event times (`invitee_start_time`, `end_time`)  
        - Email from Dropcontact enriched email array (first email)  
        - Lead’s full name  
        - LinkedIn Profile URL  
        - Person (assigned via static people ID)  
        - Website URL  
        - LinkedIn Company URL  
        - Civility as rich text  
      - Uses expressions to map values from previous nodes  
    - **Key Expressions:**  
      - Date start: `={{$node["Function"].json["payload"]["event"]["invitee_start_time"]}}`  
      - Date end: `={{$node["Function"].json["payload"]["event"]["end_time"]}}`  
      - Email: `={{$json["email"][0]["email"]}}`  
      - Full name, LinkedIn URLs, website, civility from enriched JSON  
    - **Input/Output:**  
      - Input from Dropcontact node  
      - Output is terminal (no further nodes)  
    - **Version Requirements:** n8n version supporting Notion node and API version matching credentials  
    - **Potential Failures:**  
      - Missing or incorrect Notion database ID or credentials  
      - API rate limits or connectivity issues  
      - Missing required properties in Notion database schema  
      - Expression evaluation failures if fields are missing or malformed  

---

### 3. Summary Table

| Node Name       | Node Type                | Functional Role                  | Input Node(s)     | Output Node(s) | Sticky Note                                                                                 |
|-----------------|--------------------------|---------------------------------|-------------------|----------------|---------------------------------------------------------------------------------------------|
| Calendly Trigger| calendlyTrigger          | Trigger on new Calendly event    | —                 | Dropcontact    |                                                                                             |
| Dropcontact     | dropcontact              | Verify and enrich contact data   | Calendly Trigger  | Notion        |                                                                                             |
| Notion          | notion                   | Create task in Notion database   | Dropcontact       | —              |                                                                                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Calendly Trigger node:**  
   - Type: `Calendly Trigger`  
   - Configure to listen for event `invitee.created`  
   - Use valid Calendly OAuth2 credentials  
   - Leave webhook ID blank for automatic webhook creation  
   - Position on the canvas at (460, 400) (optional for organization)

2. **Create the Dropcontact node:**  
   - Type: `Dropcontact`  
   - Connect input from Calendly Trigger’s output  
   - Set email parameter to `{{$json["payload"]["invitee"]["email"]}}`  
   - Under options, enable `siren` and set language to `"fr"`  
   - Under additional fields, map:  
     - `full_name` = `{{$json["payload"]["invitee"]["name"]}}`  
     - `last_name` = `{{$json["payload"]["invitee"]["last_name"]}}`  
     - `first_name` = `{{$json["payload"]["invitee"]["first_name"]}}`  
   - Set Dropcontact API credentials (API key)  
   - Position at (650, 400)

3. **Create the Notion node:**  
   - Type: `Notion`  
   - Connect input from Dropcontact node output  
   - Configure resource as `databasePage`  
   - Set the Notion database ID to the target task database  
   - Map properties to Notion fields:  
     - Date (range):  
       - Start: `={{$node["Function"].json["payload"]["event"]["invitee_start_time"]}}`  
       - End: `={{$node["Function"].json["payload"]["event"]["end_time"]}}`  
     - Email: `={{$json["email"][0]["email"]}}`  
     - Leads (title): `={{$json["full_name"]}}`  
     - LinkedIn Profile (URL): `={{$json["linkedin"]}}`  
     - Person (people): Static array with ID `["22ad678a-175a-405c-b504-978d7804ebb8"]`  
     - Website (URL): `={{$json["website"]}}`  
     - LinkedIn Company (URL): `={{$json["company_linkedin"]}}`  
     - Civility (rich_text): `={{$json["civility"]}}`  
   - Add Notion API credentials (OAuth2 or Integration token)  
   - Position at (850, 400)

4. **Connect nodes in order:**  
   - Calendly Trigger → Dropcontact → Notion

5. **Test the workflow:**  
   - Schedule a new meeting in Calendly to trigger the workflow  
   - Verify the enriched contact data in Dropcontact output  
   - Check the created task in Notion for correctness

---

### 5. General Notes & Resources

| Note Content                                                                 | Context or Link                                          |
|------------------------------------------------------------------------------|----------------------------------------------------------|
| Centralizes meeting bookings into Notion with verified contact enrichment.   | Workflow description                                      |
| Dropcontact enriches and verifies email and contact data for reliability.    | https://dropcontact.com/                                  |
| Notion database must have matching schema fields for successful page creation.| Notion API documentation: https://developers.notion.com/ |
| Calendly webhook setup is automatic but requires valid Calendly OAuth2 creds.| Calendly API docs: https://developer.calendly.com/       |
| Ensure API keys and tokens have correct scopes for all integrated services.  | n8n credential setup documentation                        |

---

This document fully captures the structure, logic, configuration, and reproduction steps of the described n8n workflow for transforming meeting bookings into Notion tasks with verified contact information.