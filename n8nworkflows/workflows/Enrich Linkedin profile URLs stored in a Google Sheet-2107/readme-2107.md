Enrich Linkedin profile URLs stored in a Google Sheet

https://n8nworkflows.xyz/workflows/enrich-linkedin-profile-urls-stored-in-a-google-sheet-2107


# Enrich Linkedin profile URLs stored in a Google Sheet

### 1. Workflow Overview

This workflow automates the enrichment of LinkedIn profile URLs stored in a Google Sheet by retrieving contact information, particularly email addresses, using the Prospeo.io LinkedIn Email Finder API. It is designed for users who want to augment their LinkedIn data with verified email and other profile details, then update the same Google Sheet with enriched information.

The workflow consists of the following logical blocks:

- **1.1 Scheduled Data Retrieval**  
  Periodically triggers the workflow to start the data enrichment process.

- **1.2 Data Acquisition from Google Sheets**  
  Reads LinkedIn profile URLs and associated metadata from a specified Google Sheet.

- **1.3 Conditional Filtering**  
  Filters out records with insufficient data (empty Name, Gender, Job Title, Summary but with a LinkedIn URL present) to decide which profiles to enrich.

- **1.4 API Integration to Enrich Data**  
  Sends HTTP POST requests to the Prospeo.io API with LinkedIn URLs to fetch enriched profile information.

- **1.5 Data Merging and Field Editing**  
  Merges original and retrieved data, then formats and sets the fields to prepare for updating.

- **1.6 Google Sheets Update**  
  Writes the enriched profile data back to the Google Sheet, updating existing rows based on a unique ID.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Data Retrieval

- **Overview:**  
  This block triggers the entire workflow at regular, defined intervals (every minute), automating the enrichment process.

- **Nodes Involved:**  
  - Schedule Trigger

- **Node Details:**  
  - **Schedule Trigger**  
    - **Type:** Trigger node  
    - **Configuration:** Set to trigger every minute based on the 'minutes' interval field.  
    - **Key Expressions:** None  
    - **Inputs:** None (trigger node)  
    - **Outputs:** Connects to "Get links from Google Sheet" node  
    - **Potential Failures:** Misconfiguration of schedule interval, node downtime.  
    - **Documentation:** [Schedule Trigger Node](https://docs.n8n.io/nodes/n8n-nodes-base.scheduleTrigger)

---

#### 1.2 Data Acquisition from Google Sheets

- **Overview:**  
  Reads LinkedIn profile URLs and associated metadata from a specific Google Sheet, providing the base data for enrichment.

- **Nodes Involved:**  
  - Get links from Google Sheet

- **Node Details:**  
  - **Get links from Google Sheet**  
    - **Type:** Google Sheets node (Read operation)  
    - **Configuration:**  
      - Document ID: `1ochnGSCy8V5Mz-nr51dBmugqR50m62K7d6pvbwOHewo`  
      - Sheet Name: `gid=0` (first sheet)  
    - **Credentials:** Uses OAuth2 credentials named "Omar sheet"  
    - **Key Expressions:** None (reads all rows)  
    - **Inputs:** Receives trigger from Schedule Trigger  
    - **Outputs:** Sends data to Conditional Check node  
    - **Potential Failures:** Authentication errors, permission issues, document/sheet not found, API rate limits.  
    - **Documentation:** [Google Sheets Node](https://docs.n8n.io/nodes/n8n-nodes-base.googleSheets)

---

#### 1.3 Conditional Filtering

- **Overview:**  
  Filters input data to process only rows that have an empty Name, Gender, Job Title, and Summary, but a non-empty LinkedIn URL, ensuring only incomplete profiles are enriched.

- **Nodes Involved:**  
  - Conditional Check

- **Node Details:**  
  - **Conditional Check**  
    - **Type:** If node (conditional logic)  
    - **Configuration:**  
      - Conditions combined with AND operator:  
        - Name is empty  
        - Gender is empty  
        - Job Title is empty  
        - Summary is empty  
        - LinkedIn URL is NOT empty  
    - **Key Expressions:** Uses expressions like `={{ $json.Name }}`, `={{ $json.Gender }}` etc.  
    - **Inputs:** Receives data from Google Sheets Read  
    - **Outputs:**  
      - True branch: To HTTP Request and Data Merge  
      - False branch: To No Operation node (does nothing)  
    - **Potential Failures:** Expression evaluation errors if fields are missing, unexpected data types.  
    - **Documentation:** [Conditional Node](https://docs.n8n.io/nodes/n8n-nodes-base.if)

---

#### 1.4 API Integration to Enrich Data

- **Overview:**  
  Calls the Prospeo.io LinkedIn Email Finder API to fetch enriched profile data for the filtered LinkedIn URLs.

- **Nodes Involved:**  
  - HTTP Request - Utilize Prospeo.io LinkedIn Email Finder API1

- **Node Details:**  
  - **HTTP Request - Utilize Prospeo.io LinkedIn Email Finder API1**  
    - **Type:** HTTP Request node  
    - **Configuration:**  
      - URL: `https://api.prospeo.io/linkedin-email-finder`  
      - Method: POST  
      - Headers: `X-KEY` set with API key `43b7e4f5c6558ccaa539e0e5f5778f09`  
      - Body: JSON containing `"url"` (LinkedIn URL from input) and `"id"` (profile ID)  
      - Sends both headers and body  
    - **Key Expressions:** Uses `={{ $json['Linkden URL'] }}` and `={{ $json.ID }}`  
    - **Inputs:** True branch output from Conditional Check  
    - **Outputs:** To Data Merge node  
    - **Potential Failures:**  
      - Authentication failure due to invalid or expired API key  
      - HTTP errors (timeouts, 4xx/5xx responses)  
      - Rate limiting by API provider  
      - Malformed input data causing API rejection  
    - **Documentation:** [HTTP Request Node](https://docs.n8n.io/nodes/n8n-nodes-base.httpRequest)  
    - **Sticky Note:** Explains API usage and benefits: https://prospeo.io/api/linkedin-email-finder

---

#### 1.5 Data Merging and Field Editing

- **Overview:**  
  Combines the original Google Sheet data with the API response, then maps and sets the enriched fields preparing for update.

- **Nodes Involved:**  
  - Data Merge  
  - Field Editing

- **Node Details:**  
  - **Data Merge**  
    - **Type:** Merge node  
    - **Configuration:**  
      - Mode: Combine (merge by position)  
    - **Inputs:**  
      - From Conditional Check false branch: Original data (passed through No Operation node)  
      - From HTTP Request node: API response data  
    - **Outputs:** To Field Editing  
    - **Potential Failures:** Input order mismatch, empty inputs causing incomplete merge  
    - **Documentation:** [Merge Node](https://docs.n8n.io/nodes/n8n-nodes-base.merge)  

  - **Field Editing**  
    - **Type:** Set node  
    - **Configuration:**  
      - Sets fields: Name, Gender, Email, Summary, Education, Skills, Picture, Job Title, Location, LinkedIn link, ID  
      - Values are assigned from API response JSON paths like `response.full_name`, `response.email.email`, `response.education[0].school.name` etc.  
    - **Inputs:** From Data Merge  
    - **Outputs:** To Google Sheets Update node  
    - **Potential Failures:** Missing or undefined fields in API response, array access errors, expression evaluation errors  
    - **Documentation:** [Set Node](https://docs.n8n.io/nodes/n8n-nodes-base.set)

---

#### 1.6 Google Sheets Update

- **Overview:**  
  Updates the Google Sheet with the enriched profile data, matching rows by unique ID.

- **Nodes Involved:**  
  - Update the sheet with information

- **Node Details:**  
  - **Update the sheet with information**  
    - **Type:** Google Sheets node (Update operation)  
    - **Configuration:**  
      - Document ID: `1ochnGSCy8V5Mz-nr51dBmugqR50m62K7d6pvbwOHewo`  
      - Sheet Name: `gid=0`  
      - Operation: Update  
      - Matching Columns: `ID` (used to find the row to update)  
      - Columns updated: ID, Name, Email, Gender, Skills, Picture, Summary, Location, Education, Job Title  
    - **Credentials:** Same "Omar sheet" OAuth2 credentials  
    - **Inputs:** From Field Editing  
    - **Outputs:** None (end of workflow)  
    - **Potential Failures:** Authentication/permission errors, row not found for update, API quotas, schema mismatches  
    - **Documentation:** [Google Sheets Node](https://docs.n8n.io/nodes/n8n-nodes-base.googleSheets)

---

### 3. Summary Table

| Node Name                                      | Node Type              | Functional Role                            | Input Node(s)                        | Output Node(s)                         | Sticky Note                                                                                          |
|------------------------------------------------|------------------------|--------------------------------------------|------------------------------------|---------------------------------------|----------------------------------------------------------------------------------------------------|
| Schedule Trigger                               | Trigger                | Starts workflow on schedule                 | None                               | Get links from Google Sheet            | See detailed documentation and usage [Schedule Trigger Node](https://docs.n8n.io/nodes/n8n-nodes-base.scheduleTrigger) |
| Get links from Google Sheet                     | Google Sheets          | Reads LinkedIn URLs and metadata            | Schedule Trigger                   | Conditional Check                      | See node documentation [Google Sheets Node](https://docs.n8n.io/nodes/n8n-nodes-base.googleSheets) |
| Conditional Check                              | If                     | Filters incomplete profiles with LinkedIn URL | Get links from Google Sheet        | HTTP Request, Data Merge, No Operation | See node documentation [Conditional Node](https://docs.n8n.io/nodes/n8n-nodes-base.if)              |
| HTTP Request - Utilize Prospeo.io LinkedIn Email Finder API1 | HTTP Request           | Calls API to enrich data                     | Conditional Check (true branch)    | Data Merge                            | API info and benefits: https://prospeo.io/api/linkedin-email-finder                                  |
| No Operation, do nothing                       | NoOp                   | Placeholder for filtered-out data            | Conditional Check (false branch)   | Data Merge                            | -                                                                                                  |
| Data Merge                                    | Merge                  | Combines original and API response data     | Conditional Check, HTTP Request, No Operation | Field Editing                         | See node documentation [Merge Node](https://docs.n8n.io/nodes/n8n-nodes-base.merge)                 |
| Field Editing                                 | Set                    | Maps and sets enriched fields                | Data Merge                        | Update the sheet with information     | See node documentation [Set Node](https://docs.n8n.io/nodes/n8n-nodes-base.set)                     |
| Update the sheet with information             | Google Sheets          | Updates Google Sheet with enriched data      | Field Editing                     | None                                  | See node documentation [Google Sheets Node](https://docs.n8n.io/nodes/n8n-nodes-base.googleSheets) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**  
   - Type: Schedule Trigger  
   - Set to trigger every 1 minute on the "minutes" interval.  
   - Connect its output to the next node.

2. **Create a Google Sheets node to read data**  
   - Type: Google Sheets  
   - Operation: Read  
   - Configure Document ID: `1ochnGSCy8V5Mz-nr51dBmugqR50m62K7d6pvbwOHewo`  
   - Sheet Name: `gid=0` (or your primary sheet name)  
   - Set credentials for Google Sheets OAuth2 (create or reuse credential, e.g., "Omar sheet").  
   - Connect Schedule Trigger output to this node.

3. **Create a Conditional Check (If) node**  
   - Type: If node  
   - Configure conditions with AND combinator:  
     - `Name` is empty  
     - `Gender` is empty  
     - `Job Title` is empty  
     - `Summary` is empty  
     - `LinkedIn URL` is NOT empty  
   - Connect Google Sheets Read output to this node  
   - Configure True output for profiles to be enriched  
   - Connect False output to a No Operation node.

4. **Create a No Operation node**  
   - Type: NoOp  
   - Connect the False output of Conditional Check to this node.

5. **Create an HTTP Request node**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.prospeo.io/linkedin-email-finder`  
   - Headers: Add header `X-KEY` with your API key (replace with your own key)  
   - Body Parameters (JSON):  
     - `url`: expression referencing `{{$json["Linkden URL"]}}`  
     - `id`: expression referencing `{{$json.ID}}`  
   - Send body and headers enabled  
   - Connect True output of Conditional Check to this node.

6. **Create a Merge node**  
   - Type: Merge  
   - Mode: Combine  
   - Combination mode: Merge by position  
   - Connect False output path from Conditional Check → No Operation → Merge (input 1)  
   - Connect HTTP Request output → Merge (input 2)

7. **Create a Set node for Field Editing**  
   - Type: Set  
   - Configure fields with values from API response:  
     - Name = `{{$json.response.full_name}}`  
     - Gender = `{{$json.response.gender}}`  
     - Email = `{{$json.response.email.email}}`  
     - Summary = `{{$json.response.summary}}`  
     - Education = `{{$json.response.education[0].school.name}}`  
     - Skills = `{{$json.response.skills}}`  
     - Picture = `{{$json.response.picture}}`  
     - Job Title = `{{$json.response.job_title}}`  
     - Location = `{{$json.response.location.raw}}`  
     - Linkden link = `{{$json.response.linkedin}}`  
     - ID = `{{$json.ID}}`  
   - Connect Merge output to this node.

8. **Create a Google Sheets node to update data**  
   - Type: Google Sheets  
   - Operation: Update  
   - Document ID: `1ochnGSCy8V5Mz-nr51dBmugqR50m62K7d6pvbwOHewo`  
   - Sheet Name: `gid=0`  
   - Matching Columns: `ID`  
   - Columns to update: ID, Name, Email, Gender, Skills, Picture, Summary, Location, Education, Job Title  
   - Use same Google Sheets OAuth2 credentials ("Omar sheet")  
   - Connect Set node output to this update node.

9. **Save and activate the workflow**  
   - Test with sample data and verify updates in Google Sheets.

---

### 5. General Notes & Resources

| Note Content                                                                                                          | Context or Link                                                                                                  |
|-----------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| Video walkthrough on setup and execution of this workflow                                                             | [Quick set up video](https://www.canva.com/design/DAF9a_UBxWY/xSSlSUzRdxCPtfgx9RzGSg/watch?utm_content=DAF9a_UBxWY&utm_campaign=designshare&utm_medium=link&utm_source=editor) |
| Template users must obtain and configure a valid API key for Prospeo.io LinkedIn Email Finder API                       | https://prospeo.io/api/linkedin-email-finder                                                                    |
| Google Sheets document example for linked profile URLs and enriched data                                              | [Google Sheets](https://docs.google.com/spreadsheets/d/1ochnGSCy8V5Mz-nr51dBmugqR50m62K7d6pvbwOHewo/edit?usp=sharing) |
| API usage example with cURL                                                                                            | Provided in description section above                                                                            |
| Node documentation links included in sticky notes for each primary node                                              | See respective node documentation URLs in section 2                                                             |

---

This structured reference document provides a comprehensive understanding of the workflow’s architecture, node configuration, and operational logic, enabling effective reproduction, troubleshooting, and extension.