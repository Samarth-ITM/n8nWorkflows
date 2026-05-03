Enrich Google Sheet contacts with Dropcontact

https://n8nworkflows.xyz/workflows/enrich-google-sheet-contacts-with-dropcontact-1304


# Enrich Google Sheet contacts with Dropcontact

### 1. Workflow Overview

This workflow automates the enrichment and verification of contact information stored in a Google Sheet using Dropcontact’s API, then adds the enriched contacts to a Lemlist campaign for outreach. It serves marketing, sales, or CRM teams seeking to enhance their contact lists with verified emails and detailed data, streamlining lead generation and campaign management.

The workflow is divided into three main logical blocks:
- **1.1 Input Reception:** Manual trigger and Google Sheets data retrieval.
- **1.2 Contact Enrichment:** Using Dropcontact to verify and enrich contact details.
- **1.3 Campaign Integration:** Adding enriched contacts to a Lemlist campaign.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
This block initiates the workflow manually and retrieves all contact entries from a specified Google Sheet for processing.

**Nodes Involved:**  
- On clicking 'execute' (Manual Trigger)  
- Google Sheets

**Node Details:**

- **On clicking 'execute'**  
  - *Type & Role:* Manual Trigger node, entry point for workflow execution.  
  - *Configuration:* Default manual trigger with no parameters.  
  - *Expressions/Variables:* None.  
  - *Input/Output:* No input; outputs a trigger signal to Google Sheets node.  
  - *Edge Cases:* User must manually initiate; no automatic scheduling.  
  - *Sub-workflow:* None.

- **Google Sheets**  
  - *Type & Role:* Google Sheets node to list rows from a spreadsheet.  
  - *Configuration:*  
    - Range: A:K (columns A to K).  
    - Continue on error: false (stops on errors).  
    - Sheet ID: Must be specified (currently blank; user must configure).  
    - Authentication: OAuth2 (Google Sheets OAuth2 credentials required).  
  - *Expressions:* None specific.  
  - *Input/Output:* Input from manual trigger; outputs array of rows (contacts) to Dropcontact.  
  - *Edge Cases:*  
    - Missing or incorrect Sheet ID will cause failure.  
    - OAuth2 token expiration or invalid credentials.  
    - Empty sheets result in no data passed forward.  
  - *Sub-workflow:* None.

---

#### 2.2 Contact Enrichment

**Overview:**  
This block processes each contact from Google Sheets, enriching and verifying their email and related data using Dropcontact’s API.

**Nodes Involved:**  
- Dropcontact

**Node Details:**

- **Dropcontact**  
  - *Type & Role:* Dropcontact node for email verification and contact enrichment.  
  - *Configuration:*  
    - Email input from current item’s "email" field.  
    - Options: request SIREN (company identifier), language set to French.  
    - Additional fields supplied for improved enrichment: company name, website, LinkedIn URL, full name, last name, first name (all mapped from Google Sheet columns).  
    - Credentials: Dropcontact API key configured.  
  - *Expressions:*  
    - Email: `{{$json["email"]}}`  
    - Company: `{{$json["companyName"]}}`  
    - Website: `{{$json["website"]}}`  
    - LinkedIn: `{{$json["LinkedIn"]}}`  
    - Full name: `{{$json["fullName"]}}`  
    - Last name: `{{$json["lastName"]}}`  
    - First name: `{{$json["firstName"]}}`  
  - *Input/Output:*  
    - Input: contact data from Google Sheets.  
    - Output: enriched contact data including verified email and company info to Lemlist.  
  - *Edge Cases:*  
    - API quota limits or authentication errors.  
    - Missing input email or malformed data causing no enrichment.  
    - Partial enrichment if some fields are absent.  
  - *Sub-workflow:* None.

---

#### 2.3 Campaign Integration

**Overview:**  
This block adds each enriched contact to a Lemlist campaign, enabling automated outreach.

**Nodes Involved:**  
- Lemlist

**Node Details:**

- **Lemlist**  
  - *Type & Role:* Lemlist node to add leads to a campaign.  
  - *Configuration:*  
    - Email: taken from Dropcontact node’s first email result (`{{$node["Dropcontact"].json["email"][0]["email"]}}`).  
    - Campaign ID: must be specified by the user (currently blank).  
    - Additional fields: last name, first name, company name mapped from Dropcontact output.  
    - Credentials: Lemlist API credentials configured.  
  - *Expressions:*  
    - Email: `{{$node["Dropcontact"].json["email"][0]["email"]}}`  
    - Last Name: `{{$node["Dropcontact"].json["last_name"]}}`  
    - First Name: `{{$node["Dropcontact"].json["first_name"]}}`  
    - Company Name: `{{$node["Dropcontact"].json["company"]}}`  
  - *Input/Output:*  
    - Input: enriched contact data from Dropcontact.  
    - Output: response from Lemlist API (confirmation or error).  
  - *Edge Cases:*  
    - Invalid or missing campaign ID will cause failure.  
    - API rate limits or authentication failures.  
    - No valid verified email from Dropcontact results in missing lead.  
  - *Sub-workflow:* None.

---

### 3. Summary Table

| Node Name            | Node Type              | Functional Role                     | Input Node(s)          | Output Node(s)       | Sticky Note                            |
|----------------------|------------------------|-----------------------------------|-----------------------|----------------------|--------------------------------------|
| On clicking 'execute' | Manual Trigger         | Triggers workflow manually        | -                     | Google Sheets        |                                      |
| Google Sheets        | Google Sheets          | Reads contacts from Google Sheet  | On clicking 'execute'  | Dropcontact          |                                      |
| Dropcontact          | Dropcontact API        | Enriches and verifies contacts    | Google Sheets          | Lemlist              |                                      |
| Lemlist              | Lemlist API            | Adds contacts to Lemlist campaign | Dropcontact            | -                    |                                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.

2. **Add the Manual Trigger node:**
   - Name it "On clicking 'execute'".
   - Leave default settings.

3. **Add Google Sheets node:**
   - Connect "On clicking 'execute'" → "Google Sheets".  
   - Set operation to "Read Rows" or "Lookup" (depends on n8n version).  
   - Configure:  
     - Spreadsheet ID: Enter your target Google Sheet ID.  
     - Range: "A:K" (to read columns A to K).  
     - Authentication: Setup OAuth2 credentials for Google Sheets API (create credentials in n8n with Google account).  
     - Set "Continue on fail" to false.  
     - Name node "Google Sheets".

4. **Add Dropcontact node:**
   - Connect "Google Sheets" → "Dropcontact".  
   - Configure credentials using your Dropcontact API key.  
   - Set parameters:  
     - Email: `{{$json["email"]}}` (maps from Google Sheet data).  
     - Options: enable "Siren" (company ID), language "fr" (French).  
     - Additional fields: map company, website, LinkedIn, full name, last name, first name from Google Sheets fields respectively:  
       - company: `{{$json["companyName"]}}`  
       - website: `{{$json["website"]}}`  
       - linkedin: `{{$json["LinkedIn"]}}`  
       - full_name: `{{$json["fullName"]}}`  
       - last_name: `{{$json["lastName"]}}`  
       - first_name: `{{$json["firstName"]}}`  
   - Name node "Dropcontact".

5. **Add Lemlist node:**
   - Connect "Dropcontact" → "Lemlist".  
   - Configure credentials with Lemlist API credentials.  
   - Set parameters:  
     - Email: `{{$node["Dropcontact"].json["email"][0]["email"]}}`  
     - Campaign ID: Enter the target Lemlist campaign ID.  
     - Additional fields:  
       - lastName: `{{$node["Dropcontact"].json["last_name"]}}`  
       - firstName: `{{$node["Dropcontact"].json["first_name"]}}`  
       - companyName: `{{$node["Dropcontact"].json["company"]}}`  
   - Name node "Lemlist".

6. **Save the workflow.**

7. **Run the workflow manually** using the manual trigger node to test.

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                        |
|-------------------------------------------------------------------------------------------------|--------------------------------------|
| The workflow requires valid API credentials for Google Sheets (OAuth2), Dropcontact, and Lemlist. | Credential setup in n8n must be done before execution. |
| Google Sheet ID and Lemlist Campaign ID must be set according to your environment.               | These are mandatory parameters.       |
| Dropcontact API supports enrichment beyond email, including company data (SIREN) and social links. | https://dropcontact.com/en           |
| Lemlist node adds leads directly to campaigns, streamlining outbound email processes.            | https://www.lemlist.com/api-docs     |
| Manual trigger node requires user to start the workflow; scheduling can be added if needed.      | n8n scheduling options at https://docs.n8n.io/         |

---

This documentation enables a thorough understanding, reproduction, and troubleshooting of the "Enrich Google Sheet contacts with Dropcontact" workflow in n8n.