Domain to Email Extraction using Apollo API 

https://n8nworkflows.xyz/workflows/domain-to-email-extraction-using-apollo-api--3581


# Domain to Email Extraction using Apollo API 

### 1. Workflow Overview

This workflow automates the extraction of business email contacts from company domains using the Apollo API, integrating Google Sheets as both input and output data stores. It is designed primarily for sales, marketing, business development, and recruiting professionals who need to build enriched lead lists efficiently.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Trigger and retrieval of target company domains from a Google Sheet.
- **1.2 Domain People Search:** Query Apollo API to find people associated with each domain.
- **1.3 People Data Cleanup and Looping:** Process and normalize Apollo API responses, then iterate over each person.
- **1.4 Person Profile Enrichment:** Fetch detailed profile information for each person via Apollo API.
- **1.5 Final Data Cleanup and Output:** Extract key contact fields and append them to a Google Sheet results tab.
- **1.6 Loop Control:** Manage batch processing of domains and people to respect API limits and automate sequential processing.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  Initiates the workflow manually and pulls the list of target domains from a Google Sheet tab named "Target Domains".

- **Nodes Involved:**  
  - When clicking ‘Test workflow’ (Manual Trigger)  
  - Pull Target Domains (Google Sheets)

- **Node Details:**

  - **When clicking ‘Test workflow’**  
    - Type: Manual Trigger  
    - Role: Starts the workflow on user command.  
    - Configuration: No parameters; simple manual start.  
    - Inputs: None  
    - Outputs: Triggers "Pull Target Domains" node.  
    - Edge Cases: None typical; user must manually trigger.

  - **Pull Target Domains**  
    - Type: Google Sheets  
    - Role: Reads rows from the "Target Domains" sheet (gid=0) filtering by "Status" column (filter configured but no specific filter values shown).  
    - Configuration: Reads the "Domain To Enrich" column from the sheet with document ID set to the target Google Sheet.  
    - Credentials: Uses Google Sheets OAuth2 credentials named "JKM Sheets".  
    - Inputs: Trigger from Manual Trigger node.  
    - Outputs: Passes domain data to "Loop Targets".  
    - Edge Cases:  
      - Google Sheets API quota exceeded or auth failure.  
      - Empty or malformed domain entries.  
      - Missing or incorrect document ID or sheet name.

#### 2.2 Domain People Search

- **Overview:**  
  Processes each domain individually, querying Apollo's mixed_people search API to find people associated with the domain.

- **Nodes Involved:**  
  - Loop Targets (SplitInBatches)  
  - Get People By Domain (HTTP Request)  
  - Clean Up Results (Code)

- **Node Details:**

  - **Loop Targets**  
    - Type: SplitInBatches  
    - Role: Iterates over each domain from Google Sheets one at a time to control API usage.  
    - Configuration: Default batch size (1) to process domains sequentially.  
    - Inputs: Domains from "Pull Target Domains".  
    - Outputs: Sends one domain at a time to "Get People By Domain".  
    - Edge Cases: Large domain lists may cause long execution times.

  - **Get People By Domain**  
    - Type: HTTP Request  
    - Role: Calls Apollo API endpoint `/v1/mixed_people/search` with POST method.  
    - Configuration:  
      - Body JSON includes the current domain in `q_organization_domains_list`.  
      - Requests up to 10 people per domain (`per_page: 10`).  
      - Headers include `x-api-key` (user must replace with their Apollo API key) and `Content-Type: application/json`.  
    - Inputs: Single domain from "Loop Targets".  
    - Outputs: Raw Apollo API response to "Clean Up Results".  
    - Edge Cases:  
      - API key invalid or missing.  
      - API rate limits exceeded.  
      - No people found for domain returns empty array.

  - **Clean Up Results**  
    - Type: Code (JavaScript)  
    - Role: Parses Apollo API response, extracts an array of people, and normalizes their data into structured JSON objects.  
    - Configuration: Extracts fields such as first name, last name, email, social URLs, phone, company info, and professional details.  
    - Inputs: Raw API response from "Get People By Domain".  
    - Outputs: Array of normalized people objects to "Loop Over Results".  
    - Edge Cases:  
      - Missing or malformed data fields.  
      - Unexpected API response structure causing exceptions (try-catch implemented).  
      - Empty people array results in no output items.

#### 2.3 People Data Cleanup and Looping

- **Overview:**  
  Iterates over each person extracted from the domain search results to enrich their profile data individually.

- **Nodes Involved:**  
  - Loop Over Results (SplitInBatches)  
  - Loop Targets (re-invoked)  
  - Get Person Info (HTTP Request)

- **Node Details:**

  - **Loop Over Results**  
    - Type: SplitInBatches  
    - Role: Processes each person from "Clean Up Results" one at a time to respect API rate limits.  
    - Configuration: Default batch size (1).  
    - Inputs: Normalized people array.  
    - Outputs: Two outputs:  
      - First output loops back to "Loop Targets" to process next domain after all people processed.  
      - Second output sends each person to "Get Person Info" for enrichment.  
    - Edge Cases: Large number of people per domain can increase runtime.

  - **Get Person Info**  
    - Type: HTTP Request  
    - Role: Calls Apollo API `/api/v1/people/match` endpoint to enrich individual person profiles.  
    - Configuration:  
      - URL constructed dynamically with query parameters: first name, last name, domain (extracted from "Get People By Domain" breadcrumbs).  
      - Method: POST (though typical RESTful match endpoints are GET; this matches workflow config).  
      - Headers include `x-api-key` and `Content-Type`.  
    - Inputs: Single person JSON from "Loop Over Results".  
    - Outputs: Raw enriched profile data to "Clean Up".  
    - Edge Cases:  
      - API key issues.  
      - Person not found or incomplete data returned.  
      - URL expression errors if domain data missing.

#### 2.4 Final Data Cleanup and Output

- **Overview:**  
  Extracts key contact fields from enriched person profiles and appends them to the "Results" Google Sheet.

- **Nodes Involved:**  
  - Clean Up (Code)  
  - Results To Results Sheet (Google Sheets)

- **Node Details:**

  - **Clean Up**  
    - Type: Code (JavaScript)  
    - Role: Extracts and normalizes fields from the enriched Apollo person object, such as first/last name, email, LinkedIn URL, title, social URLs, company name, headline, and photo URL.  
    - Inputs: Raw person profile from "Get Person Info".  
    - Outputs: Cleaned person data to "Results To Results Sheet".  
    - Edge Cases:  
      - Missing fields handled with default empty strings.  
      - Errors caught and logged, workflow throws error on failure.

  - **Results To Results Sheet**  
    - Type: Google Sheets  
    - Role: Appends cleaned contact data to the "Results" tab in the Google Sheet.  
    - Configuration:  
      - Maps fields: Company (from current domain), First Name, Last Name, Title, Email, LinkedIn.  
      - Document ID and sheet ID set to target Google Sheet and "Results" tab (gid=308352805).  
      - Append operation mode.  
    - Credentials: Uses Google Sheets OAuth2 credentials "JKM Sheets".  
    - Inputs: Cleaned person data from "Clean Up".  
    - Outputs: Loops back to "Loop Over Results" to continue processing.  
    - Edge Cases:  
      - Google Sheets API quota or permission errors.  
      - Data mapping errors if fields missing.

#### 2.5 Loop Control and Workflow Flow

- **Overview:**  
  Manages the iteration over domains and people, ensuring sequential processing and looping back to process all domains.

- **Nodes Involved:**  
  - Loop Targets  
  - Loop Over Results

- **Node Details:**

  - **Loop Targets**  
    - Acts as the main domain iterator.  
    - Receives domains from "Pull Target Domains" and loops through them one by one.  
    - After processing people for a domain, receives control back from "Loop Over Results" to process next domain.

  - **Loop Over Results**  
    - Iterates over people found for a domain.  
    - After processing all people, triggers "Loop Targets" to continue with next domain.

- **Edge Cases:**  
  - Infinite loops if exit conditions are not met (not applicable here as batches exhaust data).  
  - Long runtimes for large domain or people lists.  
  - API rate limits require batch size 1 to avoid throttling.

---

### 3. Summary Table

| Node Name               | Node Type         | Functional Role                              | Input Node(s)               | Output Node(s)               | Sticky Note                                                                                                         |
|-------------------------|-------------------|----------------------------------------------|-----------------------------|-----------------------------|---------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger    | Starts the workflow manually                  | None                        | Pull Target Domains          |                                                                                                                     |
| Pull Target Domains      | Google Sheets     | Reads target domains from Google Sheet        | When clicking ‘Test workflow’ | Loop Targets                | Requires Google Sheets credentials and correct document ID.                                                        |
| Loop Targets            | SplitInBatches    | Iterates over each domain one at a time       | Pull Target Domains          | Get People By Domain, Loop Targets | Controls batch size to respect API limits.                                                                         |
| Get People By Domain     | HTTP Request     | Queries Apollo API for people by domain       | Loop Targets                | Clean Up Results             | Requires Apollo API key; fetches up to 10 people per domain.                                                       |
| Clean Up Results         | Code              | Normalizes Apollo API people data              | Get People By Domain         | Loop Over Results            | Handles missing fields; logs errors on failure.                                                                    |
| Loop Over Results        | SplitInBatches    | Iterates over each person found                | Clean Up Results             | Loop Targets, Get Person Info | Batch size 1 to avoid API throttling; loops back to domains after people processed.                                 |
| Get Person Info          | HTTP Request     | Enriches individual person profile via Apollo | Loop Over Results            | Clean Up                    | Dynamic URL with person and domain info; requires API key.                                                         |
| Clean Up                | Code              | Extracts key contact fields from enriched data | Get Person Info              | Results To Results Sheet     | Handles missing fields gracefully; logs and throws on errors.                                                      |
| Results To Results Sheet | Google Sheets     | Appends enriched contact info to results sheet | Clean Up                    | Loop Over Results            | Requires Google Sheets credentials and correct document ID; appends data to "Results" tab.                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - No parameters.  
   - This node will start the workflow manually.

2. **Create Google Sheets Node to Pull Target Domains**  
   - Type: Google Sheets  
   - Operation: Read rows  
   - Document ID: Set to your Google Sheet ID containing target domains.  
   - Sheet Name: Set to the "Target Domains" tab (gid=0).  
   - Filter: Optionally filter by "Status" column if used.  
   - Credentials: Connect your Google Sheets OAuth2 credentials.  
   - Connect Manual Trigger output to this node.

3. **Create SplitInBatches Node to Loop Over Domains ("Loop Targets")**  
   - Type: SplitInBatches  
   - Batch Size: 1 (default) to process domains sequentially.  
   - Connect output of "Pull Target Domains" to this node.

4. **Create HTTP Request Node to Get People By Domain**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.apollo.io/v1/mixed_people/search`  
   - Headers:  
     - `x-api-key`: Your Apollo API key  
     - `Content-Type`: application/json  
   - Body: JSON with:  
     ```json
     {
       "q_organization_domains_list": ["{{ $json['Domain To Enrich'] }}"],
       "per_page": 10,
       "page": 1
     }
     ```  
   - Send Body: true, Specify Body: JSON  
   - Connect output of "Loop Targets" to this node.

5. **Create Code Node to Clean Up Results ("Clean Up Results")**  
   - Type: Code (JavaScript)  
   - Paste the provided JS code that extracts and normalizes people data from the Apollo response.  
   - Connect output of "Get People By Domain" to this node.

6. **Create SplitInBatches Node to Loop Over People ("Loop Over Results")**  
   - Type: SplitInBatches  
   - Batch Size: 1 to process people sequentially.  
   - Connect output of "Clean Up Results" to this node.

7. **Create HTTP Request Node to Get Person Info**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: Construct dynamically as:  
     ```
     https://api.apollo.io/api/v1/people/match?first_name={{ $json.firstName }}&last_name={{ $json.lastName }}&domain={{ $('Get People By Domain').item.json.breadcrumbs[0].value[0] }}&reveal_personal_emails=false&reveal_phone_number=false
     ```  
   - Headers:  
     - `x-api-key`: Your Apollo API key  
     - `Content-Type`: application/json  
   - Connect output of "Loop Over Results" (second output) to this node.

8. **Create Code Node to Clean Up Enriched Person Data ("Clean Up")**  
   - Type: Code (JavaScript)  
   - Paste the provided JS code that extracts key fields from the enriched person profile.  
   - Connect output of "Get Person Info" to this node.

9. **Create Google Sheets Node to Append Results ("Results To Results Sheet")**  
   - Type: Google Sheets  
   - Operation: Append  
   - Document ID: Your Google Sheet ID  
   - Sheet Name: "Results" tab (gid=308352805)  
   - Columns Mapping:  
     - Company: `={{ $('Loop Targets').item.json['Domain To Enrich'] }}`  
     - First Name: `={{ $json.firstName }}`  
     - Last Name: `={{ $json.lastName }}`  
     - Title: `={{ $json.title }}`  
     - Email: `={{ $json.email }}`  
     - Linkedin: `={{ $json.linkedinUrl }}`  
   - Credentials: Google Sheets OAuth2  
   - Connect output of "Clean Up" to this node.

10. **Connect Looping Back**  
    - Connect output of "Results To Results Sheet" back to the first output of "Loop Over Results" to continue processing next person or domain.  
    - Connect the first output of "Loop Over Results" back to the second output of "Loop Targets" to continue processing next domain.

11. **Verify Credentials and API Keys**  
    - Ensure Google Sheets OAuth2 credentials are configured and authorized.  
    - Insert your Apollo API key in both HTTP Request nodes.

12. **Test the Workflow**  
    - Add test domains in the "Target Domains" sheet.  
    - Run the workflow manually via the trigger node.  
    - Verify results populate in the "Results" sheet.

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                   |
|---------------------------------------------------------------------------------------------------------------|--------------------------------------------------|
| Workflow respects Apollo API rate limits by processing domains and people one at a time to avoid throttling.  | Important for stable API integration              |
| Apollo API may not return contact info for all domains or all employees; expect some empty results.           | Data completeness caveat                           |
| Consider legal and privacy implications when collecting and storing personal contact information.             | Compliance reminder                                |
| Workflow made with ❤️ by [Hueston](https://hueston.co/)                                                       | Project credit and branding                        |
| Google Sheets setup requires two tabs: "Target Domains" and "Results" with specified columns.                 | Setup instructions                                |
| To automate, replace manual trigger with Schedule Trigger and add status filtering in Google Sheets.          | Customization tip                                  |
| Apollo API documentation: https://apolloio.github.io/apollo-api-docs/                                         | Reference for API endpoints and parameters        |

---

This document provides a complete, structured reference for understanding, reproducing, and modifying the "Domain to Email Extraction using Apollo API" workflow in n8n. It covers all nodes, their roles, configurations, and integration details, enabling both human users and automation agents to work effectively with the workflow.