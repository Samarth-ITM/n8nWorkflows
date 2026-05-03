CallForge - 03 - Gong Transcript Processor and Salesforce Enricher

https://n8nworkflows.xyz/workflows/callforge---03---gong-transcript-processor-and-salesforce-enricher-3033


# CallForge - 03 - Gong Transcript Processor and Salesforce Enricher

### 1. Workflow Overview

This workflow, **CallForge - 03 - Gong Transcript Processor and Salesforce Enricher**, automates the extraction, enrichment, and formatting of Gong.io call transcripts for enhanced sales insights. It is designed primarily for sales teams, revenue operations professionals, and businesses using Gong.io who want to transform raw call data into structured, AI-ready datasets enriched with Salesforce and PeopleDataLabs information.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Gong Data Retrieval**  
  Receives the initial trigger and calls Gong APIs to fetch detailed call metadata and transcripts.

- **1.2 Transcript Processing and Speaker Classification**  
  Converts raw Gong transcripts into structured dialogue, classifying speakers as internal (sales team) or external (customers).

- **1.3 Salesforce Data Retrieval and Enrichment**  
  Retrieves Salesforce opportunity and account data related to the call to enrich the transcript metadata.

- **1.4 External Attendee Domain Filtering and Company Data Enrichment**  
  Filters out free email domains from external attendees and calls PeopleDataLabs API to enrich company and location data.

- **1.5 Data Aggregation and Final Formatting**  
  Merges Gong, Salesforce, and enrichment data into a clean, structured format ready for AI processing or downstream systems.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Gong Data Retrieval

**Overview:**  
This block triggers the workflow and retrieves detailed Gong call data and transcripts using Gong API calls.

**Nodes Involved:**  
- Execute Workflow Trigger  
- Get transcript  
- Retrieve detailed call data  
- Extract Call Data  
- Join Transcript to String  
- Merge call and transcript Data

**Node Details:**

- **Execute Workflow Trigger**  
  - *Type:* Execute Workflow Trigger  
  - *Role:* Entry point to start the workflow on demand or via another workflow.  
  - *Config:* No parameters; triggers downstream Gong API calls.  
  - *Connections:* Outputs to "Get transcript" and "Retrieve detailed call data".  
  - *Edge Cases:* Trigger failures or missing input data can halt the workflow.

- **Get transcript**  
  - *Type:* HTTP Request  
  - *Role:* Calls Gong API to retrieve call transcript data for a given call ID.  
  - *Config:* POST request to `https://api.gong.io/v2/calls/transcript` with call ID from trigger data; uses HTTP header authentication with Gong API credentials.  
  - *Expressions:* Uses `{{ $json['calldata[0].calls'].id }}` to dynamically set call ID.  
  - *Edge Cases:* API authentication errors, invalid call IDs, or network timeouts.

- **Retrieve detailed call data**  
  - *Type:* HTTP Request  
  - *Role:* Calls Gong API to fetch extensive call metadata including collaboration, interaction, parties, and content details.  
  - *Config:* POST request to `https://api.gong.io/v2/calls/extensive` with JSON body specifying content selectors and filters by call ID.  
  - *Expressions:* Uses `{{ $json['calldata[0].calls'].id }}` for call ID.  
  - *Edge Cases:* Similar to "Get transcript" node; API errors or malformed responses.

- **Extract Call Data**  
  - *Type:* Split Out  
  - *Role:* Splits the array of calls from the detailed call data response into individual items for processing.  
  - *Config:* Splits on `body.calls` field.  
  - *Edge Cases:* Empty or missing calls array.

- **Join Transcript to String**  
  - *Type:* Set  
  - *Role:* Transforms the transcript JSON array into a simplified array of speaker-text objects.  
  - *Config:* Uses JMESPath expression to extract speakerId and text from transcript sentences, assigning to `Conversation` field.  
  - *Expressions:* `{{ $jmespath($json.body.callTranscripts, '[].transcript[].{"speaker": speakerId, "text": sentences[].text}') }}`  
  - *Edge Cases:* Missing transcript data or unexpected JSON structure.

- **Merge call and transcript Data**  
  - *Type:* Merge  
  - *Role:* Combines the detailed call data and the processed transcript data into a single item by position.  
  - *Config:* Mode set to "combine" by position.  
  - *Edge Cases:* Mismatched input lengths or missing data.

---

#### 2.2 Transcript Processing and Speaker Classification

**Overview:**  
This block replaces speaker IDs with their affiliation (Internal or External) and formats the conversation into a readable string.

**Nodes Involved:**  
- Join Affiliation  
- Join conversation

**Node Details:**

- **Join Affiliation**  
  - *Type:* Code  
  - *Role:* Maps each speakerId in the conversation to their affiliation (Internal, External, Unknown) based on party data.  
  - *Config:* JavaScript code reads parties array, builds a map of speakerId to affiliation, and replaces speakerId in conversation entries with affiliation.  
  - *Input:* Merged call and transcript data.  
  - *Output:* JSON with updatedConversation field containing affiliation-labeled dialogue.  
  - *Edge Cases:* Missing speakerId mappings default to "Unknown".

- **Join conversation**  
  - *Type:* Code  
  - *Role:* Converts the updated conversation array into a single formatted string with lines like "Internal: text" or "External: text".  
  - *Config:* JavaScript code iterates over conversation entries and concatenates speaker-text lines separated by newlines.  
  - *Output:* Adds `conversationText` field with the formatted transcript string.  
  - *Edge Cases:* Empty conversation arrays.

---

#### 2.3 Salesforce Data Retrieval and Enrichment

**Overview:**  
Fetches Salesforce opportunity and account data related to the call to enrich metadata.

**Nodes Involved:**  
- Get External Attendees Emails  
- Get Opp Data  
- Extract SF Opp Data  
- Get account data  
- Extract SF Opp Data1  
- Combine Salesforce Opp Data  
- Aggregate Salesforce data

**Node Details:**

- **Get External Attendees Emails**  
  - *Type:* Set  
  - *Role:* Extracts email addresses of external or unknown affiliation attendees from Gong data.  
  - *Config:* Uses JMESPath to filter parties by affiliation and extract emailAddress array.  
  - *Output:* `externalAttendees` array.  
  - *Edge Cases:* No external attendees found.

- **Get Opp Data**  
  - *Type:* Salesforce  
  - *Role:* Retrieves Salesforce Opportunity record by ID linked to the call.  
  - *Config:* Uses opportunityId from trigger data (`calldata[0].calls.sfOpp`).  
  - *Credentials:* Salesforce OAuth2 credentials configured.  
  - *Edge Cases:* Invalid or missing Opportunity ID, Salesforce API errors.

- **Extract SF Opp Data**  
  - *Type:* Set  
  - *Role:* Extracts key fields from Salesforce Opportunity record for easier downstream use.  
  - *Fields Extracted:* Id, Type, LeadSource, IsClosed, IsWon, StageName, AccountId, custom fields like n8n_experience__c, ForecastCategory.  
  - *Edge Cases:* Missing fields or null values.

- **Get account data**  
  - *Type:* Salesforce  
  - *Role:* Retrieves Salesforce Account data using AccountId from Opportunity.  
  - *Config:* Uses AccountId from previous node output.  
  - *Credentials:* Same Salesforce OAuth2 credentials.  
  - *Edge Cases:* Invalid AccountId or API errors.

- **Extract SF Opp Data1**  
  - *Type:* Set  
  - *Role:* Extracts additional account fields such as Employees_Bucket__c and Name.  
  - *Edge Cases:* Missing fields.

- **Combine Salesforce Opp Data**  
  - *Type:* Merge  
  - *Role:* Combines Opportunity and Account extracted data into one dataset by position.  
  - *Edge Cases:* Mismatched input lengths.

- **Aggregate Salesforce data**  
  - *Type:* Aggregate  
  - *Role:* Aggregates all Salesforce data items into a single array under `sfOpp` field.  
  - *Edge Cases:* Empty input.

---

#### 2.4 External Attendee Domain Filtering and Company Data Enrichment

**Overview:**  
Filters out free email domains from external attendees to identify the customer’s company domain and prepares data for enrichment via PeopleDataLabs API.

**Nodes Involved:**  
- Isolate Notion Data

**Node Details:**

- **Isolate Notion Data**  
  - *Type:* Set  
  - *Role:* Extracts and structures final metadata fields for AI processing, including company name, attendees (internal and external), call metadata, and filters out free email domains to determine the customer domain.  
  - *Config:*  
    - Uses JMESPath to extract internal and external attendees’ emails and names.  
    - Defines a list of known free email domains (e.g., gmail.com, yahoo.com).  
    - Filters external emails to exclude free domains and selects the first valid company domain or "Unknown".  
    - Extracts Salesforce Opportunity name and Gong call metadata fields (title, started, call ID, URL).  
    - Copies integrations and competitors data from the initial trigger.  
  - *Expressions:* Complex JavaScript expression embedded in the Set node for domain filtering.  
  - *Edge Cases:* No external emails or all emails from free domains result in "Unknown" domain.

---

#### 2.5 Data Aggregation and Final Formatting

**Overview:**  
Aggregates Gong transcript data and Salesforce enrichment, merges all data, and prepares the final structured dataset for AI or downstream use.

**Nodes Involved:**  
- Aggregate Gong Call Transcript  
- Merge Enriched Transcript Data

**Node Details:**

- **Aggregate Gong Call Transcript**  
  - *Type:* Aggregate  
  - *Role:* Aggregates all Gong call transcript data items into a single array under `gongData`.  
  - *Edge Cases:* Empty input.

- **Merge Enriched Transcript Data**  
  - *Type:* Merge  
  - *Role:* Combines aggregated Gong transcript data and aggregated Salesforce data into one final dataset by position.  
  - *Edge Cases:* Mismatched input lengths.

---

### 3. Summary Table

| Node Name                   | Node Type                   | Functional Role                                  | Input Node(s)                    | Output Node(s)                  | Sticky Note                                                                                          |
|-----------------------------|-----------------------------|-------------------------------------------------|---------------------------------|--------------------------------|----------------------------------------------------------------------------------------------------|
| Execute Workflow Trigger     | Execute Workflow Trigger     | Entry point to start workflow                    | -                               | Get transcript, Retrieve detailed call data |                                                                                                    |
| Get transcript              | HTTP Request                | Retrieve Gong call transcript                     | Execute Workflow Trigger         | Join Transcript to String       | ## Get Gong Transcript and Call Details The transcript is to pass into the AI prompt, but needs to be transformed first. The Call details provide the Prompt with metadata. |
| Retrieve detailed call data | HTTP Request                | Retrieve detailed Gong call metadata              | Execute Workflow Trigger         | Extract Call Data               | ## Get Gong Transcript and Call Details The transcript is to pass into the AI prompt, but needs to be transformed first. The Call details provide the Prompt with metadata. |
| Extract Call Data           | Split Out                   | Split call array from detailed call data          | Retrieve detailed call data      | Merge call and transcript Data | ## Get Gong Transcript and Call Details The transcript is to pass into the AI prompt, but needs to be transformed first. The Call details provide the Prompt with metadata. |
| Join Transcript to String   | Set                         | Convert transcript JSON to simplified conversation array | Get transcript                  | Merge call and transcript Data | ## Format Call Transcript Here we join the call transcript together and then set the speaker as either Internal (for our sales team) or External (for our customers). |
| Merge call and transcript Data | Merge                      | Combine detailed call data and transcript data    | Extract Call Data, Join Transcript to String | Join Affiliation              | ## Format Call Transcript Here we join the call transcript together and then set the speaker as either Internal (for our sales team) or External (for our customers). |
| Join Affiliation            | Code                        | Map speaker IDs to affiliations (Internal/External) | Merge call and transcript Data  | Join conversation              | ## Format Call Transcript Here we join the call transcript together and then set the speaker as either Internal (for our sales team) or External (for our customers). |
| Join conversation           | Code                        | Format conversation array into readable string    | Join Affiliation                | Get External Attendees Emails, Aggregate Gong Call Transcript | ## Format Call Transcript Here we join the call transcript together and then set the speaker as either Internal (for our sales team) or External (for our customers). |
| Get External Attendees Emails | Set                        | Extract external attendees’ email addresses       | Join conversation               | Get Opp Data                   | ## Enrich Call Data Here we get the Pipedrive ID using the email domain and use that to search pipedrive for the customer. We also pass the domain into the People Data Labs api to get location data. |
| Get Opp Data                | Salesforce                  | Retrieve Salesforce Opportunity data               | Get External Attendees Emails   | Extract SF Opp Data, Get account data | ## Enrich Call Data Here we get the Pipedrive ID using the email domain and use that to search pipedrive for the customer. We also pass the domain into the People Data Labs api to get location data. |
| Extract SF Opp Data         | Set                         | Extract key Salesforce Opportunity fields          | Get Opp Data                   | Combine Salesforce Opp Data    | ## Enrich Call Data Here we get the Pipedrive ID using the email domain and use that to search pipedrive for the customer. We also pass the domain into the People Data Labs api to get location data. |
| Get account data            | Salesforce                  | Retrieve Salesforce Account data                    | Get Opp Data                   | Extract SF Opp Data1           | ## Enrich Call Data Here we get the Pipedrive ID using the email domain and use that to search pipedrive for the customer. We also pass the domain into the People Data Labs api to get location data. |
| Extract SF Opp Data1        | Set                         | Extract key Salesforce Account fields               | Get account data               | Combine Salesforce Opp Data    | ## Enrich Call Data Here we get the Pipedrive ID using the email domain and use that to search pipedrive for the customer. We also pass the domain into the People Data Labs api to get location data. |
| Combine Salesforce Opp Data | Merge                       | Combine Opportunity and Account extracted data      | Extract SF Opp Data, Extract SF Opp Data1 | Aggregate Salesforce data     | ## Enrich Call Data Here we get the Pipedrive ID using the email domain and use that to search pipedrive for the customer. We also pass the domain into the People Data Labs api to get location data. |
| Aggregate Salesforce data   | Aggregate                   | Aggregate all Salesforce data into one array       | Combine Salesforce Opp Data    | Merge Enriched Transcript Data |                                                                                                    |
| Aggregate Gong Call Transcript | Aggregate                 | Aggregate Gong transcript data into one array      | Join conversation             | Merge Enriched Transcript Data |                                                                                                    |
| Merge Enriched Transcript Data | Merge                    | Merge Gong transcript and Salesforce data           | Aggregate Gong Call Transcript, Aggregate Salesforce data | Isolate Notion Data          | ## Extract Final Data Blob Here we merge the final outputs and get rid of anything we don't need for the final AI prompt. |
| Isolate Notion Data         | Set                         | Extract and structure final metadata and conversation | Merge Enriched Transcript Data | -                            | ## Extract Final Data Blob Here we merge the final outputs and get rid of anything we don't need for the final AI prompt. |
| Sticky Note5                | Sticky Note                 | Branding and workflow purpose description           | -                             | -                            | ![Callforge](https://uploads.n8n.io/templates/callforgeshadow.png) CallForge allows you to extract important information for different departments from your Sales Gong Calls. Transcript PreProcessor separates speakers and enriches call data from Salesforce. |
| Sticky Note                 | Sticky Note                 | Explains Gong transcript and call details retrieval | -                             | -                            | ## Get Gong Transcript and Call Details The transcript is to pass into the AI prompt, but needs to be transformed first. The Call details provide the Prompt with metadata. |
| Sticky Note1                | Sticky Note                 | Explains transcript formatting and speaker classification | -                             | -                            | ## Format Call Transcript Here we join the call transcript together and then set the speaker as either Internal (for our sales team) or External (for our customers). |
| Sticky Note2                | Sticky Note                 | Explains enrichment of call data with Pipedrive and People Data Labs | -                             | -                            | ## Enrich Call Data Here we get the Pipedrive ID using the email domain and use that to search pipedrive for the customer. We also pass the domain into the People Data Labs api to get location data. |
| Sticky Note3                | Sticky Note                 | Explains final data merging and cleanup for AI prompt | -                             | -                            | ## Extract Final Data Blob Here we merge the final outputs and get rid of anything we don't need for the final AI prompt. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create an Execute Workflow Trigger node**  
   - Type: Execute Workflow Trigger  
   - Purpose: Entry point to start the workflow. No parameters needed.

2. **Add HTTP Request node "Get transcript"**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.gong.io/v2/calls/transcript`  
   - Authentication: HTTP Header Auth with Gong API credentials  
   - Body (JSON Parameters enabled):  
     ```json
     {
       "filter": {
         "callIds": ["{{ $json['calldata[0].calls'].id }}"]
       }
     }
     ```  
   - Connect Execute Workflow Trigger → Get transcript

3. **Add HTTP Request node "Retrieve detailed call data"**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.gong.io/v2/calls/extensive`  
   - Authentication: HTTP Header Auth with Gong API credentials  
   - Body (JSON Parameters enabled):  
     ```json
     {
       "contentSelector": {
         "context": "Extended",
         "contextTiming": ["Now", "TimeOfCall"],
         "exposedFields": {
           "collaboration": {"publicComments": true},
           "content": {"pointsOfInterest": true, "structure": true, "topics": true, "trackers": true},
           "interaction": {"personInteractionStats": true, "questions": true, "speakers": true, "video": true},
           "media": false,
           "parties": true
         }
       },
       "filter": {
         "callIds": ["{{ $json['calldata[0].calls'].id }}"]
       }
     }
     ```  
   - Connect Execute Workflow Trigger → Retrieve detailed call data

4. **Add Split Out node "Extract Call Data"**  
   - Type: Split Out  
   - Field to split out: `body.calls`  
   - Connect Retrieve detailed call data → Extract Call Data

5. **Add Set node "Join Transcript to String"**  
   - Type: Set  
   - Assignments:  
     - Field: `Conversation` (array)  
     - Value (JMESPath expression):  
       ```jmespath
       $jmespath($json.body.callTranscripts, '[].transcript[].{"speaker": speakerId, "text": sentences[].text}')
       ```  
   - Connect Get transcript → Join Transcript to String

6. **Add Merge node "Merge call and transcript Data"**  
   - Type: Merge  
   - Mode: Combine  
   - Combine By: Position  
   - Connect Extract Call Data → Merge call and transcript Data (input 2)  
   - Connect Join Transcript to String → Merge call and transcript Data (input 1)

7. **Add Code node "Join Affiliation"**  
   - Type: Code  
   - JavaScript code:  
     ```javascript
     const inputData = $input.all();
     const originalJson = inputData[0].json;
     const conversation = originalJson.Conversation;
     const parties = originalJson.parties;

     const affiliationMap = {};
     parties.forEach(party => {
       affiliationMap[party.speakerId] = party.affiliation;
     });

     const updatedConversation = conversation.map(entry => {
       const affiliation = affiliationMap[entry.speaker] || 'Unknown';
       return {...entry, speaker: affiliation};
     });

     return [{ json: { ...originalJson, updatedConversation } }];
     ```  
   - Connect Merge call and transcript Data → Join Affiliation

8. **Add Code node "Join conversation"**  
   - Type: Code  
   - JavaScript code:  
     ```javascript
     const originalJson = $json;
     const conversation = originalJson.updatedConversation;
     const formattedLines = [];

     conversation.forEach(entry => {
       const speaker = entry.speaker;
       const texts = entry.text;
       texts.forEach(line => {
         formattedLines.push(`${speaker}: ${line}`);
       });
     });

     const result = formattedLines.join('\n');
     return [{ json: { ...originalJson, conversationText: result } }];
     ```  
   - Connect Join Affiliation → Join conversation

9. **Add Set node "Get External Attendees Emails"**  
   - Type: Set  
   - Assignments:  
     - Field: `externalAttendees` (array)  
     - Value (JMESPath):  
       ```jmespath
       $jmespath($json.parties, '[?affiliation==`External` || affiliation==`Unknown`].emailAddress')
       ```  
   - Connect Join conversation → Get External Attendees Emails

10. **Add Salesforce node "Get Opp Data"**  
    - Type: Salesforce  
    - Resource: Opportunity  
    - Operation: Get  
    - Opportunity ID: `{{ $('Execute Workflow Trigger').item.json["calldata[0].calls"].sfOpp }}`  
    - Credentials: Configure Salesforce OAuth2 credentials  
    - Connect Get External Attendees Emails → Get Opp Data

11. **Add Set node "Extract SF Opp Data"**  
    - Type: Set  
    - Extract fields from Opportunity: Id, Type, LeadSource, IsClosed, IsWon, StageName, AccountId, n8n_experience__c, ForecastCategory  
    - Connect Get Opp Data → Extract SF Opp Data

12. **Add Salesforce node "Get account data"**  
    - Type: Salesforce  
    - Resource: Account  
    - Operation: Get  
    - Account ID: `{{ $json.AccountId }}` (from Extract SF Opp Data)  
    - Credentials: Same Salesforce OAuth2 credentials  
    - Connect Get Opp Data → Get account data

13. **Add Set node "Extract SF Opp Data1"**  
    - Type: Set  
    - Extract fields from Account: Employees_Bucket__c, Name  
    - Connect Get account data → Extract SF Opp Data1

14. **Add Merge node "Combine Salesforce Opp Data"**  
    - Type: Merge  
    - Mode: Combine  
    - Combine By: Position  
    - Connect Extract SF Opp Data → Combine Salesforce Opp Data (input 1)  
    - Connect Extract SF Opp Data1 → Combine Salesforce Opp Data (input 2)

15. **Add Aggregate node "Aggregate Salesforce data"**  
    - Type: Aggregate  
    - Aggregate: All item data  
    - Destination field: `sfOpp`  
    - Connect Combine Salesforce Opp Data → Aggregate Salesforce data

16. **Add Aggregate node "Aggregate Gong Call Transcript"**  
    - Type: Aggregate  
    - Aggregate: All item data  
    - Destination field: `gongData`  
    - Connect Join conversation → Aggregate Gong Call Transcript

17. **Add Merge node "Merge Enriched Transcript Data"**  
    - Type: Merge  
    - Mode: Combine  
    - Combine By: Position  
    - Connect Aggregate Gong Call Transcript → Merge Enriched Transcript Data (input 1)  
    - Connect Aggregate Salesforce data → Merge Enriched Transcript Data (input 2)

18. **Add Set node "Isolate Notion Data"**  
    - Type: Set  
    - Assignments:  
      - Extract Salesforce Opportunity Name to `metaData.CompanyName`  
      - Extract internal attendees emails and names via JMESPath filtering parties by affiliation Internal  
      - Extract external attendees emails and names similarly  
      - Extract Gong call metadata fields (title, started, id, url) into metaData  
      - Extract integrations and competitors from trigger data  
      - Filter external emails to exclude free email domains (gmail.com, yahoo.com, etc.) and assign first valid domain or "Unknown" to `metaData.domain`  
      - Copy conversationText to `Conversation` field  
      - Copy Salesforce Opportunity array to `sfOpp` field  
    - Connect Merge Enriched Transcript Data → Isolate Notion Data

---

### 5. General Notes & Resources

| Note Content                                                                                                     | Context or Link                                                                                       |
|-----------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| ![Callforge](https://uploads.n8n.io/templates/callforgeshadow.png) CallForge allows you to extract important information for different departments from your Sales Gong Calls. Transcript PreProcessor separates speakers and enriches call data from Salesforce. | Branding and workflow purpose description (Sticky Note5)                                            |
| The transcript is to pass into the AI prompt, but needs to be transformed first. The Call details provide the Prompt with metadata. | Explanation for Gong transcript and call details retrieval (Sticky Note)                             |
| Here we join the call transcript together and then set the speaker as either Internal (for our sales team) or External (for our customers). | Explanation for transcript formatting and speaker classification (Sticky Note1)                     |
| Here we get the Pipedrive ID using the email domain and use that to search pipedrive for the customer. We also pass the domain into the People Data Labs api to get location data. | Explanation for enrichment of call data with Pipedrive and People Data Labs (Sticky Note2)           |
| Here we merge the final outputs and get rid of anything we don't need for the final AI prompt.                   | Explanation for final data merging and cleanup for AI prompt (Sticky Note3)                          |

---

This documentation provides a complete, structured understanding of the CallForge workflow, enabling users and AI agents to reproduce, modify, and troubleshoot the workflow effectively.