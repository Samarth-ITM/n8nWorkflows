Track and Score Contact Engagement with Zoho CRM, PDL, News & Reddit

https://n8nworkflows.xyz/workflows/track-and-score-contact-engagement-with-zoho-crm--pdl--news---reddit-11700


# Track and Score Contact Engagement with Zoho CRM, PDL, News & Reddit

### 1. Workflow Overview

This workflow, titled **"Zoho CRM - Social Media Engagement Tracker"**, is designed to track and score social media engagement for contacts stored in Zoho CRM. It targets sales, marketing, and customer success teams who want to enrich contact data with social profiles, monitor public mentions across news and Reddit, and automatically create opportunities based on engagement levels.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Contact Retrieval:** Triggered by a Zoho CRM webhook when a contact is created or updated; retrieves full contact details from Zoho CRM.
- **1.2 Contact Data Preparation:** Extracts and formats key contact fields and builds a keyword for social mention searches.
- **1.3 Social Profile Enrichment:** Uses People Data Labs (PDL) API to enrich contact data with social profiles; conditionally proceeds only if profiles are found.
- **1.4 Public Mention Mining:** Queries GNews and Reddit APIs for public mentions of the contact using the constructed keyword.
- **1.5 Scoring and Opportunity Check:** Combines mentions from news and Reddit, calculates an engagement score, and checks if it exceeds a threshold.
- **1.6 CRM Actions for High Engagement:** If engagement is high, creates a new deal (opportunity) in Zoho CRM, adds contact and account details, creates a note logging social intelligence, and updates the contact record with enriched data and scores.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Contact Retrieval

- **Overview:**  
  This block receives webhook triggers from Zoho CRM when a contact is created or updated. It fetches the full contact details from Zoho CRM using the contact ID provided by the webhook.

- **Nodes Involved:**  
  - Zoho CRM New Contact Webhook  
  - Get Contact By ID

- **Node Details:**

  - **Zoho CRM New Contact Webhook**  
    - Type: Webhook  
    - Role: Entry point triggered by Zoho CRM on contact creation/update.  
    - Configuration: HTTP POST method, path `zoho-crm-new-contact`.  
    - Inputs: External HTTP request from Zoho CRM.  
    - Outputs: JSON payload containing contact ID.  
    - Edge Cases: Webhook misconfiguration, missing contact ID in payload.

  - **Get Contact By ID**  
    - Type: Zoho CRM node  
    - Role: Retrieves full contact details from Zoho CRM using the contact ID from webhook.  
    - Configuration: Resource = Contact, Operation = Get, Contact ID from webhook JSON body.  
    - Inputs: Output from webhook node.  
    - Outputs: Full contact JSON data.  
    - Credentials: Requires Zoho OAuth2 credentials.  
    - Edge Cases: Invalid contact ID, API rate limits, authentication errors.

---

#### 1.2 Contact Data Preparation

- **Overview:**  
  Extracts and normalizes key contact fields (first name, last name, company, email, IDs) and constructs a keyword string used for social mention searches.

- **Nodes Involved:**  
  - Prepare Contact Data

- **Node Details:**

  - **Prepare Contact Data**  
    - Type: Function  
    - Role: Normalizes contact data fields and builds a search keyword combining first name, last name, and company.  
    - Configuration: Custom JavaScript function extracting fields with fallback keys, concatenating names and company for keyword.  
    - Inputs: Contact JSON from "Get Contact By ID".  
    - Outputs: JSON with normalized fields: `first_name`, `last_name`, `full_name`, `company`, `email`, `keyword`, `zoho_contact_id`, `zoho_company_id`.  
    - Edge Cases: Missing fields, null or undefined values.

---

#### 1.3 Social Profile Enrichment

- **Overview:**  
  Enriches contact data by querying People Data Labs API for social profiles. If no profiles are found, the workflow stops further enrichment.

- **Nodes Involved:**  
  - PDL Enrichment  
  - IF Social Profiles Found  
  - Build Social Summary

- **Node Details:**

  - **PDL Enrichment**  
    - Type: HTTP Request  
    - Role: Calls People Data Labs API to enrich person data with social profiles.  
    - Configuration: GET request to `https://api.peopledatalabs.com/v5/person/enrich` with query parameters: name, email, company from prepared data; header includes API key.  
    - Inputs: Output from "Prepare Contact Data".  
    - Outputs: JSON with enrichment data including social profiles.  
    - Edge Cases: API key invalid, rate limits, network errors.

  - **IF Social Profiles Found**  
    - Type: If  
    - Role: Checks if the enrichment response contains any social profiles (`data.profiles.length > 0`).  
    - Configuration: Condition on length of profiles array.  
    - Inputs: Output from "PDL Enrichment".  
    - Outputs: True branch proceeds to "Build Social Summary"; false branch stops enrichment.  
    - Edge Cases: Missing or malformed response data.

  - **Build Social Summary**  
    - Type: Function  
    - Role: Constructs a summary string of social profile URLs (LinkedIn, Twitter, Facebook, GitHub) and identifies primary LinkedIn URL.  
    - Configuration: Custom JavaScript function that concatenates available profile URLs separated by " | ".  
    - Inputs: Enrichment data from PDL.  
    - Outputs: JSON with `social_summary` string and `primary_linkedin` URL.  
    - Edge Cases: Missing profile URLs.

---

#### 1.4 Public Mention Mining

- **Overview:**  
  Searches public mentions of the contact on news (GNews) and Reddit using the keyword generated from contact data.

- **Nodes Involved:**  
  - GNews Mentions  
  - Reddit Mentions

- **Node Details:**

  - **GNews Mentions**  
    - Type: HTTP Request  
    - Role: Queries GNews API for news articles matching the keyword.  
    - Configuration: GET request to `https://gnews.io/api/v4/search` with query parameters: `q` (keyword), `lang` = "en", and API key.  
    - Inputs: Output from "Build Social Summary" (keyword from "Prepare Contact Data").  
    - Outputs: JSON with news articles.  
    - Edge Cases: API key invalid, rate limits, no results.

  - **Reddit Mentions**  
    - Type: HTTP Request  
    - Role: Queries Reddit search API for posts matching the keyword.  
    - Configuration: GET request to `https://www.reddit.com/search.json` with query parameter `q` (keyword), custom User-Agent header.  
    - Inputs: Output from "GNews Mentions".  
    - Outputs: JSON with Reddit posts.  
    - Edge Cases: Reddit API rate limits, no results, network errors.

---

#### 1.5 Scoring and Opportunity Check

- **Overview:**  
  Combines news and Reddit mentions, calculates an engagement score based on mention counts, likes, and comments, and determines if the score exceeds a threshold for high engagement.

- **Nodes Involved:**  
  - Combine Mentions & Score  
  - IF High Engagement

- **Node Details:**

  - **Combine Mentions & Score**  
    - Type: Function  
    - Role: Merges news and Reddit mentions into a single array; calculates engagement score using weighted metrics (news mentions count * 3, Reddit likes * 0.5, comments * 1).  
    - Configuration: Custom JavaScript function parsing input JSON, mapping mentions, and computing score and mention count.  
    - Inputs: Output from "Reddit Mentions".  
    - Outputs: JSON with combined mentions, `engagement_score`, and `mention_count`.  
    - Edge Cases: Missing or malformed input data.

  - **IF High Engagement**  
    - Type: If  
    - Role: Checks if engagement score is greater than or equal to 200 to trigger opportunity creation.  
    - Configuration: Numeric condition on `engagement_score >= 200`.  
    - Inputs: Output from "Combine Mentions & Score".  
    - Outputs: True branch triggers CRM deal creation and contact update; false branch updates contact only.  
    - Edge Cases: Missing score value.

---

#### 1.6 CRM Actions for High Engagement

- **Overview:**  
  For contacts with high engagement, creates a new deal (opportunity) in Zoho CRM, associates contact and account details, creates a note logging social intelligence, and updates the contact record with social profiles, engagement score, status, and mention counts.

- **Nodes Involved:**  
  - Zoho CRM Create Deal  
  - Add Contact and Account Details In Created Deal  
  - Create Note  
  - Zoho CRM Update Contact

- **Node Details:**

  - **Zoho CRM Create Deal**  
    - Type: Zoho CRM node  
    - Role: Creates a new deal in Zoho CRM with stage "Qualification" and a name based on contact full name.  
    - Configuration: Resource = Deal, Stage = Qualification, Deal Name = "Social Opportunity - [Full Name]".  
    - Inputs: True branch from "IF High Engagement".  
    - Outputs: JSON with created deal details.  
    - Credentials: Zoho OAuth2 credentials required.  
    - Edge Cases: API errors, authentication failures.

  - **Add Contact and Account Details In Created Deal**  
    - Type: HTTP Request  
    - Role: Updates the newly created deal to associate it with the contact and account IDs.  
    - Configuration: PUT request to Zoho CRM Deals endpoint with JSON body linking Contact_Name and Account_Name by IDs.  
    - Inputs: Output from "Zoho CRM Create Deal".  
    - Outputs: Updated deal JSON.  
    - Credentials: Zoho OAuth2 credentials required.  
    - Edge Cases: Invalid deal ID, API errors.

  - **Create Note**  
    - Type: HTTP Request  
    - Role: Creates a note in Zoho CRM linked to the deal, logging engagement score and mention count.  
    - Configuration: POST request to Zoho CRM Notes endpoint with JSON body containing note title, content, parent deal ID, and module.  
    - Inputs: Output from "Add Contact and Account Details In Created Deal".  
    - Outputs: Created note JSON.  
    - Credentials: Zoho OAuth2 credentials required.  
    - Edge Cases: API errors, invalid parent ID.

  - **Zoho CRM Update Contact**  
    - Type: Zoho CRM node  
    - Role: Updates the contact record with social profiles (concatenated URLs), engagement score, social status (Hot/Warm/Low), and mention counts.  
    - Configuration: Resource = Contact, Operation = Update, Contact ID from prepared data, updates custom fields: Social_Profiles, Social_Status, Engagement_Score, Mentions_Counts.  
    - Inputs: Both true and false branches from "IF High Engagement" (always updates contact).  
    - Credentials: Zoho OAuth2 credentials required.  
    - Edge Cases: Missing contact ID, API errors.

---

### 3. Summary Table

| Node Name                           | Node Type           | Functional Role                              | Input Node(s)                  | Output Node(s)                             | Sticky Note                                                                                                   |
|-----------------------------------|---------------------|----------------------------------------------|-------------------------------|--------------------------------------------|---------------------------------------------------------------------------------------------------------------|
| Zoho CRM New Contact Webhook       | Webhook             | Entry point triggered by Zoho CRM webhook    | -                             | Get Contact By ID                          | ## Zoho CRM – Social Media Engagement Tracker<br>Workflow triggers from a Zoho CRM webhook whenever a Contact is created/updated. It fetches the Contact details, enriches social profiles via People Data Labs, checks for public mentions (News + Reddit), calculates an engagement score, and updates CRM fields.<br><br>Setup steps: Add Zoho OAuth2 Credentials, Set Webhook URL inside Zoho CRM → Contacts module, Add PDL API Key + GNews Key, Add custom fields exist on Contacts: Social_Profiles, Engagement_Score, Mentions_Counts , Social_Status, Test with a Contact that has public profiles + mentions |
| Get Contact By ID                  | Zoho CRM            | Fetch full contact details from Zoho CRM     | Zoho CRM New Contact Webhook  | Prepare Contact Data                       | Contact Intake & Preparation<br>Receives Zoho ContactID and loads full details from CRM. Extracts name, email, company, CRM Contact and Company IDs and builds the search keyword. |
| Prepare Contact Data              | Function            | Normalize contact fields and build keyword   | Get Contact By ID              | PDL Enrichment                            | Contact Intake & Preparation (same as above)                                                                  |
| PDL Enrichment                   | HTTP Request        | Enrich contact with social profiles from PDL | Prepare Contact Data           | IF Social Profiles Found                   | Enrichment<br>Fetches social profile metadata and stops early if none found.                                   |
| IF Social Profiles Found          | If                  | Check if social profiles exist                | PDL Enrichment                | Build Social Summary                       | Enrichment (same as above)                                                                                      |
| Build Social Summary             | Function            | Summarize social profile URLs                 | IF Social Profiles Found       | GNews Mentions                            | Public Mention Mining<br>Queries news + Reddit using the combined keyword derived from the contact.            |
| GNews Mentions                  | HTTP Request        | Search news mentions via GNews API            | Build Social Summary           | Reddit Mentions                           | Public Mention Mining (same as above)                                                                           |
| Reddit Mentions                 | HTTP Request        | Search Reddit mentions                         | GNews Mentions                | Combine Mentions & Score                   | Public Mention Mining (same as above)                                                                           |
| Combine Mentions & Score         | Function            | Merge mentions, calculate engagement score    | Reddit Mentions               | IF High Engagement                        | Scoring & Opportunity Check<br>Merges mentions, scores & identifies high-value opportunities.                  |
| IF High Engagement              | If                  | Check if engagement score exceeds threshold   | Combine Mentions & Score       | Zoho CRM Create Deal, Zoho CRM Update Contact | Scoring & Opportunity Check (same as above)                                                                     |
| Zoho CRM Create Deal             | Zoho CRM            | Create new deal/opportunity in Zoho CRM       | IF High Engagement (true)      | Add Contact and Account Details In Created Deal | CRM Actions (If High Engagement Found)<br>Creates a new opportunity and logs all social intelligence to the CRM. |
| Add Contact and Account Details In Created Deal | HTTP Request        | Link contact and account to created deal      | Zoho CRM Create Deal           | Create Note                               | CRM Actions (If High Engagement Found) (same as above)                                                         |
| Create Note                     | HTTP Request        | Create note logging engagement details        | Add Contact and Account Details In Created Deal | -                                        | CRM Actions (If High Engagement Found) (same as above)                                                         |
| Zoho CRM Update Contact          | Zoho CRM            | Update contact with social profiles and scores | IF High Engagement (both branches) | -                                        | Add Details in Zoho CRM<br>Updates Social Profiles, Engagement Score, Status, and Mention Count.                |
| Sticky Note                     | Sticky Note         | Documentation and overview                     | -                             | -                                          | See content in Sticky Note node                                                                                 |
| Sticky Note1                    | Sticky Note         | Documentation for Contact Intake & Preparation | -                             | -                                          | See content in Sticky Note1 node                                                                                |
| Sticky Note2                    | Sticky Note         | Documentation for Enrichment                   | -                             | -                                          | See content in Sticky Note2 node                                                                                |
| Sticky Note3                    | Sticky Note         | Documentation for Public Mention Mining        | -                             | -                                          | See content in Sticky Note3 node                                                                                |
| Sticky Note4                    | Sticky Note         | Documentation for Scoring & Opportunity Check  | -                             | -                                          | See content in Sticky Note4 node                                                                                |
| Sticky Note5                    | Sticky Note         | Documentation for CRM Actions                   | -                             | -                                          | See content in Sticky Note5 node                                                                                |
| Sticky Note6                    | Sticky Note         | Documentation for CRM Contact Update            | -                             | -                                          | See content in Sticky Note6 node                                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node: "Zoho CRM New Contact Webhook"**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `zoho-crm-new-contact`  
   - Response Mode: Last Node  
   - Purpose: Receive webhook from Zoho CRM on contact create/update.

2. **Create Zoho CRM Node: "Get Contact By ID"**  
   - Type: Zoho CRM  
   - Resource: Contact  
   - Operation: Get  
   - Contact ID: `={{ $json.body.id }}` (from webhook)  
   - Credentials: Configure Zoho OAuth2 credentials  
   - Connect output of webhook to this node.

3. **Create Function Node: "Prepare Contact Data"**  
   - Type: Function  
   - Code: Extract `First_Name`, `Last_Name`, `Account_Name.name` or `Company`, `Email`, and IDs from input JSON.  
   - Build `full_name` and `keyword` by concatenating first name, last name, and company.  
   - Output JSON fields: `first_name`, `last_name`, `full_name`, `company`, `email`, `keyword`, `zoho_contact_id`, `zoho_company_id`.  
   - Connect output of "Get Contact By ID" to this node.

4. **Create HTTP Request Node: "PDL Enrichment"**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://api.peopledatalabs.com/v5/person/enrich`  
   - Query Parameters:  
     - `name` = `={{ $json.full_name }}`  
     - `email` = `={{ $json.email }}`  
     - `company` = `={{ $json.company }}`  
   - Header: `x-api-key` with your People Data Labs API key  
   - Response Format: JSON  
   - Connect output of "Prepare Contact Data" to this node.

5. **Create If Node: "IF Social Profiles Found"**  
   - Type: If  
   - Condition: Check if `{{$json.data?.profiles?.length}} > 0`  
   - Connect output of "PDL Enrichment" to this node.

6. **Create Function Node: "Build Social Summary"**  
   - Type: Function  
   - Code: Collect URLs from `linkedin_url`, `twitter_url`, `facebook_url`, `github_url` fields if present; join with " | ".  
   - Output fields: `social_summary`, `primary_linkedin` (LinkedIn URL or empty).  
   - Connect True output of "IF Social Profiles Found" to this node.

7. **Create HTTP Request Node: "GNews Mentions"**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://gnews.io/api/v4/search`  
   - Query Parameters:  
     - `q` = `={{ $('Prepare Contact Data').item.json.keyword }}`  
     - `lang` = `en`  
     - `apikey` = Your GNews API key  
   - Connect output of "Build Social Summary" to this node.

8. **Create HTTP Request Node: "Reddit Mentions"**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://www.reddit.com/search.json`  
   - Query Parameter: `q` = `={{ $('Prepare Contact Data').item.json.keyword }}`  
   - Header: `User-Agent` = `n8n-social-intel-workflow/1.0`  
   - Connect output of "GNews Mentions" to this node.

9. **Create Function Node: "Combine Mentions & Score"**  
   - Type: Function  
   - Code:  
     - Parse news articles and Reddit posts from inputs.  
     - Map to unified mention objects with source, title, snippet, url, type, likes, comments, shares.  
     - Calculate engagement score: news mentions * 3 + Reddit likes * 0.5 + Reddit comments * 1.  
     - Output combined mentions array, engagement score, mention count.  
   - Connect output of "Reddit Mentions" to this node.

10. **Create If Node: "IF High Engagement"**  
    - Type: If  
    - Condition: `{{$json.engagement_score}} >= 200`  
    - Connect output of "Combine Mentions & Score" to this node.

11. **Create Zoho CRM Node: "Zoho CRM Create Deal"**  
    - Type: Zoho CRM  
    - Resource: Deal  
    - Operation: Create  
    - Stage: Qualification  
    - Deal Name: `Social Opportunity - {{$('Prepare Contact Data').item.json.full_name || 'Unknown'}}`  
    - Credentials: Zoho OAuth2  
    - Connect True output of "IF High Engagement" to this node.

12. **Create HTTP Request Node: "Add Contact and Account Details In Created Deal"**  
    - Type: HTTP Request  
    - Method: PUT  
    - URL: `https://www.zohoapis.com/crm/v2/Deals/{{$json.id}}`  
    - Body (JSON):  
      ```json
      {
        "data": [
          {
            "Contact_Name": { "id": "={{ $('Prepare Contact Data').item.json.zoho_contact_id }}" },
            "Account_Name": { "id": "={{ $('Prepare Contact Data').item.json.zoho_company_id }}" }
          }
        ]
      }
      ```  
    - Authentication: Zoho OAuth2  
    - Connect output of "Zoho CRM Create Deal" to this node.

13. **Create HTTP Request Node: "Create Note"**  
    - Type: HTTP Request  
    - Method: POST  
    - URL: `https://www.zohoapis.com/crm/v2/Notes`  
    - Body (JSON):  
      ```json
      {
        "data": [
          {
            "Note_Title": "Social Intelligence Log",
            "Note_Content": "Auto-note: Engagement Score:{{$('Combine Mentions & Score').item.json.engagement_score}},Mentions: {{$('Combine Mentions & Score').item.json.mention_count}}",
            "Parent_Id": { "id": "={{ $('Zoho CRM Create Deal').item.json.id }}" },
            "se_module": "Deals"
          }
        ]
      }
      ```  
    - Authentication: Zoho OAuth2  
    - Connect output of "Add Contact and Account Details In Created Deal" to this node.

14. **Create Zoho CRM Node: "Zoho CRM Update Contact"**  
    - Type: Zoho CRM  
    - Resource: Contact  
    - Operation: Update  
    - Contact ID: `={{ $('Prepare Contact Data').item.json.zoho_contact_id }}`  
    - Update Fields:  
      - Custom Fields:  
        - Social_Profiles: Concatenate URLs from PDL profiles separated by newlines.  
        - Social_Status: "Hot" if engagement_score >= 50, "Warm" if >= 20, else "Low".  
        - Engagement_Score: Rounded engagement_score.  
        - Mentions_Counts: mention_count.  
    - Credentials: Zoho OAuth2  
    - Connect False output of "IF High Engagement" to this node (update contact even if no deal created).  
    - Also connect True output branch after "IF High Engagement" (parallel update).

---

This completes the full reproduction of the workflow. Credentials for Zoho OAuth2, People Data Labs API key, and GNews API key must be configured in n8n prior to execution. Custom fields (`Social_Profiles`, `Engagement_Score`, `Mentions_Counts`, `Social_Status`) must exist in Zoho CRM Contacts module.

---

This structured documentation enables understanding, modification, and re-creation of the workflow without requiring the original JSON.