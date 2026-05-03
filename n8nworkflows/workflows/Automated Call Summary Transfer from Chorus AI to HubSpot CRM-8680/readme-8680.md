Automated Call Summary Transfer from Chorus AI to HubSpot CRM

https://n8nworkflows.xyz/workflows/automated-call-summary-transfer-from-chorus-ai-to-hubspot-crm-8680


# Automated Call Summary Transfer from Chorus AI to HubSpot CRM

### 1. Workflow Overview

This workflow automates the transfer of call engagement summaries from Chorus AI into HubSpot CRM by creating notes linked to corresponding companies. It is designed to run periodically (hourly) or can be triggered manually, fetching Chorus AI engagements from the previous day, filtering out those without summaries, and checking for matching companies in HubSpot. For each matching company, it verifies if a note with the engagement summary already exists to avoid duplicates before creating a new note.

The workflow is logically divided into four main blocks:

- **1.1 Trigger & Input Reception:** Defines how the workflow starts (scheduled or manual trigger).
- **1.2 Chorus AI Engagement Fetch & Filtering:** Retrieves Chorus AI engagements with pagination and filters out entries lacking summaries.
- **1.3 HubSpot Company & Note Search:** For each filtered engagement, searches HubSpot for a matching company and checks if a corresponding note already exists.
- **1.4 Note Creation & Looping:** Creates a detailed note in HubSpot if none exists for the engagement, then continues processing remaining engagements.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger & Input Reception

**Overview:**  
Defines the entry points for the workflow: scheduled execution every hour or manual triggering.

**Nodes Involved:**  
- Run every hour  
- When clicking ‘Execute workflow’  
- Sticky Note2  

**Node Details:**  

- **Run every hour**  
  - Type: Schedule Trigger  
  - Configuration: Triggers the workflow every hour using interval scheduling.  
  - Inputs: None  
  - Outputs: Triggers "Get Chorus Engagement per last day" node  
  - Edge cases: Scheduler failures are rare but possible if server time misconfigured.

- **When clicking ‘Execute workflow’**  
  - Type: Manual Trigger  
  - Configuration: Allows manual execution via UI button.  
  - Inputs: None  
  - Outputs: Same as Schedule Trigger, triggers "Get Chorus Engagement per last day"  
  - Edge cases: User-initiated, no technical failures expected, but user errors possible if run too frequently.

- **Sticky Note2**  
  - Type: Sticky Note  
  - Content: Explains the trigger mechanisms in the workflow.  
  - Inputs/Outputs: None  

---

#### 2.2 Chorus AI Engagement Fetch & Filtering

**Overview:**  
Fetches Chorus AI engagements from the past day using paginated API calls, merges all pages, and filters out engagements missing a 'meeting_summary'.

**Nodes Involved:**  
- Get Chorus Engagement per last day  
- Sticky Note  
- Merge paginated engagements  
- Filter items with empty meeting_summary  

**Node Details:**  

- **Get Chorus Engagement per last day**  
  - Type: HTTP Request  
  - Configuration:  
    - URL: Chorus AI engagements endpoint (`https://chorus.ai/v3/engagements`)  
    - Auth: HTTP Header Auth using Chorus.AI API Token credential  
    - Query Parameter: `min_date` set to 24 hours before current UTC time to fetch recent engagements.  
    - Pagination: Uses continuation key to fetch up to 15 pages with 250ms intervals, stops when no engagements are returned.  
  - Inputs: Trigger nodes  
  - Outputs: JSON with paginated engagement data  
  - Edge cases: API token expiration, network timeouts, pagination failures, empty responses.

- **Sticky Note**  
  - Type: Sticky Note  
  - Content: Describes this block’s purpose to load Chorus AI engagements with pagination and filter those without summaries.  

- **Merge paginated engagements**  
  - Type: Code  
  - Configuration: Custom JavaScript concatenates all engagement arrays from paginated responses into a single array for downstream processing.  
  - Inputs: Multiple paginated HTTP responses  
  - Outputs: Combined array of engagements  
  - Edge cases: Unexpected data structures, empty pages, runtime errors in JS code.

- **Filter items with empty meeting_summary**  
  - Type: Filter  
  - Configuration: Filters out any engagement where the `meeting_summary` field is empty or missing.  
  - Inputs: Merged engagements array  
  - Outputs: Only engagements having non-empty summaries  
  - Edge cases: Missing `meeting_summary` fields or incorrect data types might cause false negatives.

---

#### 2.3 HubSpot Company & Note Search

**Overview:**  
For each filtered engagement, looks up the corresponding company in HubSpot by exact name match. If found, searches for existing notes containing the engagement ID to prevent duplicate notes.

**Nodes Involved:**  
- Loop Over Engagements  
- Search Company In HubSpot By Name  
- If company exists  
- Search notes  
- If note not exist  
- Skip, if company not found  
- Skip, if note already exists  
- Sticky Note1  

**Node Details:**  

- **Loop Over Engagements**  
  - Type: SplitInBatches  
  - Configuration: Processes engagements one by one or in batches (default batch size).  
  - Inputs: Filtered engagements  
  - Outputs: Single engagement item per iteration  
  - Edge cases: Large batch sizes could lead to timeouts or rate limits.

- **Search Company In HubSpot By Name**  
  - Type: HTTP Request  
  - Configuration:  
    - URL: HubSpot CRM Search API for companies  
    - Method: POST with JSON body filtering companies where `name` equals engagement’s `account_name` exactly  
    - Auth: HubSpot App Token credential  
    - Retry enabled with 3-second wait on failure  
  - Inputs: Engagement item from loop  
  - Outputs: Search results with company details if found  
  - Edge cases: API rate limits, token expiration, company not found.

- **If company exists**  
  - Type: If  
  - Configuration: Checks if the search results contain at least one company by verifying presence of `results[0].id`.  
  - Inputs: Search Company results  
  - Outputs:  
    - True: Continue to "Search notes"  
    - False: "Skip, if company not found"  
  - Edge cases: Unexpected API responses or empty arrays.

- **Search notes**  
  - Type: HTTP Request  
  - Configuration:  
    - URL: HubSpot CRM Notes Search API  
    - Method: POST with JSON body filtering notes that contain the engagement ID in the note body and are associated with the found company ID  
    - Auth: HubSpot App Token credential  
    - Retry enabled with 3-second wait on failure  
  - Inputs: Company search results + current engagement  
  - Outputs: Note search results  
  - Edge cases: Rate limits, inconsistent note data.

- **If note not exist**  
  - Type: If  
  - Configuration: Checks if the notes search returned an empty array (`results` is empty).  
  - Inputs: Note search results  
  - Outputs:  
    - True: Proceed to create note payload  
    - False: "Skip, if note already exists"  
  - Edge cases: API inconsistencies.

- **Skip, if company not found**  
  - Type: NoOp (no operation)  
  - Configuration: Acts as a bypass when no matching company is found, loops back to next engagement.  
  - Inputs: False branch from "If company exists"  
  - Outputs: Loops to next engagement  
  - Edge cases: None.

- **Skip, if note already exists**  
  - Type: NoOp  
  - Configuration: Bypass node when note already exists, loops to next engagement.  
  - Inputs: False branch from "If note not exist"  
  - Outputs: Loops to next engagement  
  - Edge cases: None.

- **Sticky Note1**  
  - Type: Sticky Note  
  - Content: Explains the HubSpot search logic for companies and notes to avoid duplicates.  

---

#### 2.4 Note Creation & Looping

**Overview:**  
Constructs a formatted note payload with call date/time, summary, and link, then creates the note in HubSpot associated with the matched company. After creating or skipping, continues processing remaining engagements.

**Nodes Involved:**  
- Create Note Payload  
- Create Note  
- Continue  

**Node Details:**  

- **Create Note Payload**  
  - Type: Code  
  - Configuration:  
    - Extracts the engagement’s date_time (epoch seconds), converts to local date/time string  
    - Formats an HTML note body embedding call date/time, summary, call recording link, and engagement ID  
    - Constructs JSON payload including associations to the HubSpot company ID  
  - Inputs: Engagement data + company search data  
  - Outputs: JSON payload for note creation  
  - Edge cases: Date conversion errors, missing fields, encoding issues.

- **Create Note**  
  - Type: HTTP Request  
  - Configuration:  
    - URL: HubSpot CRM Notes API for note creation  
    - Method: POST with JSON payload from previous node  
    - Auth: HubSpot App Token credential  
  - Inputs: Note payload JSON  
  - Outputs: HubSpot API response confirming note creation  
  - Edge cases: API failures, token expiration, validation errors on payload.

- **Continue**  
  - Type: NoOp  
  - Configuration: Acts as a loop controller to process next engagement item.  
  - Inputs: Output from note creation or skip nodes  
  - Outputs: Loops back to "Loop Over Engagements" node  
  - Edge cases: None.

---

### 3. Summary Table

| Node Name                       | Node Type            | Functional Role                                      | Input Node(s)                         | Output Node(s)                      | Sticky Note                                                                                           |
|--------------------------------|----------------------|-----------------------------------------------------|-------------------------------------|-----------------------------------|-----------------------------------------------------------------------------------------------------|
| Run every hour                 | Schedule Trigger     | Scheduled workflow start every hour                  | None                                | Get Chorus Engagement per last day | Sticky Note2 explains triggers                                                                     |
| When clicking ‘Execute workflow’| Manual Trigger       | Manual workflow start                                | None                                | Get Chorus Engagement per last day | Sticky Note2 explains triggers                                                                     |
| Sticky Note2                   | Sticky Note          | Explains trigger mechanisms                          | None                                | None                              | ## Triggers\nHere we are declaring how this workflow can be started                                |
| Get Chorus Engagement per last day | HTTP Request        | Fetch Chorus AI engagements from last day            | Run every hour, Manual Trigger      | Merge paginated engagements       | Sticky Note describes Chorus AI engagement fetch                                                   |
| Sticky Note                   | Sticky Note          | Explains Chorus AI engagement fetching logic        | None                                | None                              | ## Chorus.AI\nThis part of code will ask Chorus AI about new engagements per last 1 day and load it with pagination and filter items which not contains information about summary. |
| Merge paginated engagements    | Code                 | Merge paginated engagement responses into one array | Get Chorus Engagement per last day  | Filter items with empty meeting_summary |                                                                                                     |
| Filter items with empty meeting_summary | Filter               | Filter out engagements without meeting summaries     | Merge paginated engagements          | Loop Over Engagements             |                                                                                                     |
| Loop Over Engagements          | SplitInBatches       | Iterate over each filtered engagement                | Filter items with empty meeting_summary | Search Company In HubSpot By Name |                                                                                                     |
| Search Company In HubSpot By Name | HTTP Request        | Search HubSpot company by exact name                 | Loop Over Engagements               | If company exists                 | Sticky Note1 explains HubSpot company and note search logic                                        |
| If company exists              | If                   | Check if HubSpot company found                        | Search Company In HubSpot By Name   | Search notes, Skip, if company not found |                                                                                                     |
| Search notes                  | HTTP Request         | Search notes associated with company containing engagement ID | If company exists                  | If note not exist                 |                                                                                                     |
| If note not exist             | If                   | Check if note for engagement already exists          | Search notes                       | Create Note Payload, Skip, if note already exists |                                                                                                     |
| Skip, if company not found    | NoOp                  | Skip processing when company not found               | If company exists (false branch)    | Loop Over Engagements             |                                                                                                     |
| Skip, if note already exists  | NoOp                  | Skip processing when note already exists              | If note not exist (false branch)    | Loop Over Engagements             |                                                                                                     |
| Create Note Payload           | Code                  | Prepare formatted note payload for HubSpot           | If note not exist                   | Create Note                      |                                                                                                     |
| Create Note                  | HTTP Request          | Create note in HubSpot CRM                            | Create Note Payload                | Continue                        |                                                                                                     |
| Continue                     | NoOp                   | Loop control to continue processing next engagement  | Create Note, Skip nodes             | Loop Over Engagements             |                                                                                                     |
| Sticky Note1                 | Sticky Note            | Explains HubSpot company and note search logic       | None                              | None                            | ## HubSpot \nHere we will search company in HubSpot by full equal name between Chorus.AI account_name and HubSpot Company Name.\nIf matched company exists, we will check if current engagement already present in notes.\nIf such engagement not present in notes, we creating it. |

---

### 4. Reproducing the Workflow from Scratch

1. **Setup credentials:**  
   - Create HTTP Header Auth credential with Chorus.AI API Token.  
   - Create HubSpot App Token credential with appropriate access.

2. **Create Trigger nodes:**  
   - Add a **Schedule Trigger** node: Configure to run every hour.  
   - Add a **Manual Trigger** node: For manual execution.

3. **Add Chorus AI Engagement Retrieval:**  
   - Add an **HTTP Request** node named "Get Chorus Engagement per last day".  
   - Set method to GET and URL to `https://chorus.ai/v3/engagements`.  
   - Add query parameter `min_date` with expression: `{{$now.minus(1, 'day').toUTC().toISO()}}`.  
   - Use HTTP Header Auth with Chorus.AI credential.  
   - Enable pagination: Use the continuation key from response body to paginate up to 15 pages with 250ms interval. Stop when no engagements returned.

4. **Merge paginated engagements:**  
   - Add **Code** node "Merge paginated engagements".  
   - JavaScript to concatenate all engagements arrays from paginated responses into a single array:  
     ```js
     let engagements = [];
     for (const item of $input.all()) {
       engagements = engagements.concat(item.json.engagements);
     }
     return engagements;
     ```

5. **Filter engagements with summaries:**  
   - Add **Filter** node "Filter items with empty meeting_summary".  
   - Condition: `meeting_summary` is not empty.

6. **Batch processing:**  
   - Add **SplitInBatches** node "Loop Over Engagements" connected to filter output. Default batch size is fine.

7. **Search HubSpot company:**  
   - Add **HTTP Request** node "Search Company In HubSpot By Name".  
   - Method: POST  
   - URL: `https://api.hubapi.com/crm/v3/objects/companies/search`  
   - Body (JSON):  
     ```json
     {
       "filterGroups": [{
         "filters": [{
           "propertyName": "name",
           "operator": "EQ",
           "value": "={{ $json.account_name }}"
         }]
       }],
       "properties": ["name", "domain", "hs_object_id"],
       "limit": 1
     }
     ```  
   - Use HubSpot App Token credential.  
   - Enable retry on fail with 3 seconds wait.

8. **Check if company exists:**  
   - Add **If** node "If company exists".  
   - Condition: Check if `results[0].id` exists (operator: exists).

9. **If company found, search notes:**  
   - Add **HTTP Request** node "Search notes".  
   - Method: POST  
   - URL: `https://api.hubapi.com/crm/v3/objects/notes/search`  
   - Body (JSON):  
     ```json
     {
       "filterGroups": [{
         "filters": [
           {
             "propertyName": "hs_note_body",
             "operator": "CONTAINS_TOKEN",
             "value": "={{ $('Loop Over Engagements').item.json.engagement_id }}"
           },
           {
             "propertyName": "associations.company",
             "operator": "EQ",
             "value": "={{ $json.results[0].id }}"
           }
         ]
       }],
       "properties": ["hs_note_body","hs_timestamp"],
       "limit": 25
     }
     ```  
   - Use HubSpot App Token credential.  
   - Enable retry on fail with 3 seconds wait.

10. **Check if note exists:**  
    - Add **If** node "If note not exist".  
    - Condition: `results` array is empty.

11. **Add NoOp nodes for skipping:**  
    - Add **NoOp** node "Skip, if company not found". Connect from false branch of "If company exists".  
    - Add **NoOp** node "Skip, if note already exists". Connect from false branch of "If note not exist".

12. **Create Note Payload:**  
    - Add **Code** node "Create Note Payload".  
    - JavaScript code:  
      ```js
      const callSecondsDateTime = $('Loop Over Engagements').first().json.date_time;
      var callDateTime = new Date(callSecondsDateTime * 1000);

      const callSummary = $('Loop Over Engagements').first().json.meeting_summary;
      const callLink = $('Loop Over Engagements').first().json.url;
      const callId = $('Loop Over Engagements').first().json.engagement_id;

      var note_body = `
      <b>Call Date/Time: ${callDateTime.toLocaleString()}</b> 
      <br><br>
      <b>Summary body:</b><br> ${callSummary}"
      <br><br>
      <b>Link to call recording:</b> <a href="${callLink}" target="_blank">${callLink}</a>
      <br>
      <b>Chorus Engagement ID:</b> ${callId}
      `;

      return {
        "json": {
          "properties": {
            "hs_timestamp": callDateTime.toISOString(),
            "hs_note_body": note_body,
          },
          "associations": [
            {
              "to": {
                "id": $('Search Company In HubSpot By Name').first().json.results[0].id
              },
              "types": [
                {
                  "associationCategory": "HUBSPOT_DEFINED",
                  "associationTypeId": 190
                }
              ]
            }
          ]
        }  
      };
      ```

13. **Create Note in HubSpot:**  
    - Add **HTTP Request** node "Create Note".  
    - Method: POST  
    - URL: `https://api.hubspot.com/crm/v3/objects/notes`  
    - Body: Use JSON from "Create Note Payload" node.  
    - Use HubSpot App Token credential.

14. **Loop continuation:**  
    - Add **NoOp** node "Continue".  
    - Connect from "Create Note" and from both NoOp skip nodes to "Continue".  
    - Connect "Continue" back to "Loop Over Engagements" to process next batch item.

15. **Connections summary:**  
    - Trigger nodes → Get Chorus Engagement per last day → Merge paginated engagements → Filter → Loop Over Engagements → Search Company → If company exists → Search notes → If note not exist → Create Note Payload → Create Note → Continue → Loop Over Engagements (loop).  
    - False branches handled by NoOp skip nodes looping back.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                      | Context or Link                                  |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------|
| Workflow automates syncing Chorus AI call summaries into HubSpot as notes linked to companies.                                                                   | Workflow purpose                                 |
| Uses robust pagination handling for Chorus AI API to ensure complete data retrieval.                                                                              | Chorus AI API docs                               |
| HubSpot API calls use app token authentication with retry on failure and delay to handle rate limits gracefully.                                                | HubSpot API docs                                 |
| Uses exact match on company name between Chorus AI and HubSpot, which can be sensitive to naming differences. Consider enhancements for fuzzy matching.         | Potential improvement                             |
| Note body includes HTML formatting with date/time localization and clickable links to call recordings.                                                          | HubSpot notes rich text support                   |
| Designed with looping and skip logic to avoid duplicate notes and handle missing companies gracefully.                                                            | Workflow resilience                               |
| Sticky notes embedded in workflow explain the purpose of each major block for maintainability and clarity.                                                       | Workflow documentation style                      |

---

**Disclaimer:**  
The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This process strictly adheres to applicable content policies and does not contain any illegal, offensive, or protected content. All data processed is legal and publicly accessible.