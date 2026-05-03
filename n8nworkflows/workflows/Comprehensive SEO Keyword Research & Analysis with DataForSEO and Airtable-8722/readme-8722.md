Comprehensive SEO Keyword Research & Analysis with DataForSEO and Airtable

https://n8nworkflows.xyz/workflows/comprehensive-seo-keyword-research---analysis-with-dataforseo-and-airtable-8722


# Comprehensive SEO Keyword Research & Analysis with DataForSEO and Airtable

---

### 1. Workflow Overview

This workflow performs comprehensive SEO keyword research and analysis by integrating DataForSEO API endpoints with Airtable as the data storage and management platform. It automates the retrieval of primary keywords from Airtable, requests various keyword data and SERP information from DataForSEO, and writes enriched keyword variations and SERP results back into Airtable for further SEO content ideation and strategy.

The workflow is logically divided into the following functional blocks:

- **1.1 Input Reception and Initialization:** Receive trigger from Airtable, fetch primary keyword and parameters, and set fields for API requests.
- **1.2 Keyword Data Retrieval:** Query multiple DataForSEO endpoints to get related keywords, keyword suggestions, keyword ideas, autocomplete keywords, SERP results, and subtopics.
- **1.3 Data Processing and Preparation:** Split API responses into individual items, set and format fields for each keyword type and SERP data.
- **1.4 Data Insertion into Airtable:** Insert processed keyword variations and SERP data into respective Airtable tables, primarily into a master keyword variations table.
- **1.5 Workflow Control and Status Update:** Update Airtable record status to mark keyword research as complete.
- **1.6 Documentation and Setup Guidance:** Sticky notes provide setup instructions, usage notes, and API documentation references.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Initialization

- **Overview:**  
  This block triggers the workflow via a webhook connected to Airtable automation. It retrieves the primary keyword record and related parameters from Airtable and prepares data fields for downstream API requests.

- **Nodes Involved:**  
  - Webhook  
  - Set Airtable Fields  
  - Get Primary Keyword  
  - Set Fields for API Request  
  - Sticky Note (Setup Instructions)

- **Node Details:**  

  - **Webhook**  
    - *Type:* Webhook (HTTP Trigger)  
    - *Role:* Entry point; receives POST requests triggered by Airtable automation with a record ID parameter.  
    - *Config:* Path set to a unique webhook ID, accepts POST method.  
    - *Inputs:* HTTP POST with query parameter `recordID`.  
    - *Outputs:* Forwards to "Set Airtable Fields".  
    - *Edge cases:* Failure if webhook URL changes or Airtable automation misconfigured.

  - **Set Airtable Fields**  
    - *Type:* Set node  
    - *Role:* Sets workflow variables including Airtable base ID, record ID, and table IDs for various Airtable tables used in the workflow.  
    - *Config:* Hardcoded Airtable base and table IDs; record ID dynamically set from webhook query parameter.  
    - *Inputs:* From Webhook.  
    - *Outputs:* To "Get Primary Keyword".  
    - *Edge cases:* Incorrect base or table IDs cause Airtable API failures.

  - **Get Primary Keyword**  
    - *Type:* Airtable node (Read record)  
    - *Role:* Fetches the record containing the primary keyword and parameters (location, language, limit, depth) from Airtable.  
    - *Config:* Uses the base and table IDs from "Set Airtable Fields"; record ID is dynamic from webhook.  
    - *Inputs:* From "Set Airtable Fields".  
    - *Outputs:* To "Set Fields for API Request".  
    - *Edge cases:* Record not found or API auth errors.

  - **Set Fields for API Request**  
    - *Type:* Set node  
    - *Role:* Extracts and assigns API request parameters (Primary Keyword, Location, Language, Limit, Depth) for subsequent API calls.  
    - *Config:* Uses expressions to map fields from "Get Primary Keyword" JSON.  
    - *Inputs:* From "Get Primary Keyword".  
    - *Outputs:* To multiple API request nodes and final Airtable update node.  
    - *Edge cases:* Missing or malformed input fields could cause API request errors.

  - **Sticky Note (Setup Instructions)**  
    - *Type:* Sticky Note  
    - *Role:* Contains detailed setup instructions for Airtable base copying, automation configuration, and webhook setup.  
    - *Content:* Setup guide with Airtable base copy link and automation script code.  
    - *No inputs or outputs.*

---

#### 2.2 Keyword Data Retrieval from DataForSEO API

- **Overview:**  
  This block performs multiple HTTP POST requests to different DataForSEO API endpoints to collect diverse keyword-related data and SERP insights.

- **Nodes Involved:**  
  - Related API Request  
  - KW Suggestions API Request  
  - KW Ideas API Request  
  - Autocomplete API Request  
  - Serp API Request1  
  - Generate Subtopics API Request  
  - Sticky Notes (descriptions for each API request)

- **Node Details:**  

  - **Related API Request**  
    - *Type:* HTTP Request  
    - *Role:* Fetches related keywords for the primary keyword.  
    - *Config:* POST to DataForSEO related_keywords endpoint; uses dynamic JSON body with primary keyword, location, language, limit, depth; HTTP Basic Authentication with DataForSEO credentials.  
    - *Inputs:* From "Set Fields for API Request".  
    - *Outputs:* To "Split Out Result Items".  
    - *Edge cases:* API auth failure, rate limiting, invalid parameters.

  - **KW Suggestions API Request**  
    - *Type:* HTTP Request  
    - *Role:* Retrieves keyword suggestions excluding synonyms.  
    - *Config:* POST to keyword_suggestions endpoint; dynamic JSON body; HTTP Basic Auth.  
    - *Inputs:* From "Set Fields for API Request".  
    - *Outputs:* To "Split Out Suggested KWs".  
    - *Edge cases:* Same as above.

  - **KW Ideas API Request**  
    - *Type:* HTTP Request  
    - *Role:* Obtains keyword ideas including close variants.  
    - *Config:* POST to keyword_ideas endpoint; dynamic JSON body; HTTP Basic Auth.  
    - *Inputs:* From "Set Fields for API Request".  
    - *Outputs:* To "Split Out KW Ideas".  
    - *Edge cases:* Same as above.

  - **Autocomplete API Request**  
    - *Type:* HTTP Request  
    - *Role:* Gets autocomplete suggestions from Google SERP.  
    - *Config:* POST to serp/google/autocomplete/live/advanced endpoint; includes client parameter; HTTP Basic Auth.  
    - *Inputs:* From "Set Fields for API Request".  
    - *Outputs:* To "Split Out Autocomplete".  
    - *Edge cases:* Same as above.

  - **Serp API Request1**  
    - *Type:* HTTP Request  
    - *Role:* Retrieves organic search results and People Also Ask data from Google SERP.  
    - *Config:* POST to serp/google/organic/live/advanced endpoint; fixed depth 25 and people also ask click depth 1; HTTP Basic Auth.  
    - *Inputs:* From "Set Fields for API Request".  
    - *Outputs:* To "Split Out SERP".  
    - *Edge cases:* Same as above.

  - **Generate Subtopics API Request**  
    - *Type:* HTTP Request  
    - *Role:* Generates subtopics for content ideation based on the primary keyword.  
    - *Config:* POST to content_generation/generate_sub_topics/live endpoint; HTTP Basic Auth.  
    - *Inputs:* From "Set Fields for API Request".  
    - *Outputs:* To "Split Out Subtopics".  
    - *Edge cases:* Same as above.

  - **Sticky Notes**  
    - Provide explanations for each API request node, including notes about hardcoded parameters and API usage guidelines.

---

#### 2.3 Data Processing and Preparation

- **Overview:**  
  This block processes the API responses by splitting arrays into individual items, extracting relevant fields, and formatting data for Airtable insertion.

- **Nodes Involved:**  
  - Split Out Result Items  
  - Split Out Suggested KWs  
  - Split Out KW Ideas  
  - Split Out Autocomplete  
  - Split Out SERP  
  - Split Out People Also Ask  
  - Split Out Subtopics  
  - Set KW Related Fields  
  - Set KW Suggestion Fields  
  - Set Keyword Ideas Fields  
  - Set Autocomplete Fields  
  - Set SERP Fields  
  - Set PAA Fileds  
  - Set Generate Subtopics Fields

- **Node Details:**  

  - **Split Out Nodes**  
    - *Type:* SplitOut  
    - *Role:* Convert API response arrays (items) into individual workflow items for granular processing and Airtable insertion.  
    - *Config:* Each splits on specific JSON arrays containing keyword or SERP items.  
    - *Inputs:* From respective API Request nodes.  
    - *Outputs:* To corresponding "Set Fields" nodes.  
    - *Edge cases:* Empty or malformed response arrays cause no output; handle with error catching upstream.

  - **Set Fields Nodes**  
    - *Type:* Set nodes  
    - *Role:* Map and rename API response fields to standardized Airtable column names; convert numeric fields; assign keyword type tags.  
    - *Config:* Use expressions to extract nested JSON fields such as keyword text, search volume (MSV), competition, CPC, keyword difficulty, search intent, etc. Assign a "type" field for classification.  
    - *Inputs:* From corresponding Split Out nodes.  
    - *Outputs:* To Airtable insertion nodes.  
    - *Edge cases:* Missing fields or unexpected data structures can cause expression evaluation errors.

---

#### 2.4 Data Insertion into Airtable

- **Overview:**  
  This block creates new records in Airtable tables for all processed keyword variations and SERP results, enriching the "Master All KW Variations" table and SERP results table.

- **Nodes Involved:**  
  - Add Related KWs to Master Table  
  - Add KW Suggestions to Master table  
  - Add KW Ideas to Master table  
  - Add Autocomplete to Master table  
  - Create SERPS  
  - Add PAA to Master Table  
  - Add PAA to Master Table1  
  - Airtable (Update Primary Keywords record)  
  - Sticky Notes (indicating Airtable insertion steps)

- **Node Details:**  

  - **Add Related KWs to Master Table**  
    - *Type:* Airtable (Create)  
    - *Role:* Inserts related keywords into master keyword variations table with relevant metrics and metadata.  
    - *Config:* Uses base and table IDs from "Set Airtable Fields"; maps fields from "Set KW Related Fields".  
    - *Inputs:* From "Set KW Related Fields".  
    - *Outputs:* None.  
    - *Edge cases:* Airtable API rate limits, invalid field mapping.

  - **Add KW Suggestions to Master table**  
    - *Type:* Airtable (Create)  
    - *Role:* Inserts keyword suggestions into master table.  
    - *Inputs:* From "Set KW Suggestion Fields".  
    - *Edge cases:* Same as above.

  - **Add KW Ideas to Master table**  
    - *Type:* Airtable (Create)  
    - *Role:* Inserts keyword ideas into master table.  
    - *Inputs:* From "Set Keyword Ideas Fields".  
    - *Edge cases:* Same as above.

  - **Add Autocomplete to Master table**  
    - *Type:* Airtable (Create)  
    - *Role:* Inserts autocomplete keyword suggestions.  
    - *Inputs:* From "Set Autocomplete Fields".  
    - *Edge cases:* Same as above.

  - **Create SERPS**  
    - *Type:* Airtable (Create)  
    - *Role:* Creates records for organic search results with detailed SERP info.  
    - *Inputs:* From "Set SERP Fields".  
    - *Edge cases:* Same as above.

  - **Add PAA to Master Table and Add PAA to Master Table1**  
    - *Type:* Airtable (Create)  
    - *Role:* Inserts People Also Ask questions and generated subtopics into master table.  
    - *Inputs:* From "Set PAA Fileds" and "Set Generate Subtopics Fields" respectively.  
    - *Edge cases:* Same as above.

  - **Airtable (Update Primary Keywords record)**  
    - *Type:* Airtable (Update)  
    - *Role:* Updates the original Primary Keywords record's "Trigger" field to "KW Research Complete" to mark workflow completion.  
    - *Inputs:* From "Set Fields for API Request".  
    - *Edge cases:* Record update failures, concurrency issues.

  - **Sticky Notes**  
    - Indicate the purpose of each insertion step and table usage.

---

#### 2.5 Workflow Control and Status Update

- **Overview:**  
  This final block updates the Airtable record to indicate that the keyword research process has completed successfully.

- **Nodes Involved:**  
  - Airtable (Update Primary Keywords record)

- **Node Details:**  

  - See details in 2.4 above.

---

### 3. Summary Table

| Node Name                     | Node Type            | Functional Role                          | Input Node(s)                | Output Node(s)                        | Sticky Note                                                    |
|-------------------------------|----------------------|----------------------------------------|-----------------------------|-------------------------------------|---------------------------------------------------------------|
| Webhook                       | Webhook              | Entry trigger from Airtable             | -                           | Set Airtable Fields                  |                                                               |
| Set Airtable Fields           | Set                  | Set Airtable and workflow parameters    | Webhook                     | Get Primary Keyword                  | Setup instructions for Airtable automation and base setup     |
| Get Primary Keyword           | Airtable             | Fetch primary keyword record             | Set Airtable Fields          | Set Fields for API Request           |                                                               |
| Set Fields for API Request    | Set                  | Prepare parameters for API calls         | Get Primary Keyword          | Multiple API Requests, Airtable update |                                                               |
| Related API Request           | HTTP Request         | Request related keywords                 | Set Fields for API Request   | Split Out Result Items               | Related Keywords API call description                          |
| Split Out Result Items        | SplitOut             | Split related keywords array             | Related API Request          | Set KW Related Fields                |                                                               |
| Set KW Related Fields         | Set                  | Map related keywords fields              | Split Out Result Items       | Add Related KWs to Master Table      |                                                               |
| Add Related KWs to Master Table| Airtable            | Insert related keywords into Airtable   | Set KW Related Fields        | -                                   | Add to Master All KW Variations table                          |
| KW Suggestions API Request    | HTTP Request         | Request keyword suggestions              | Set Fields for API Request   | Split Out Suggested KWs              | Keyword Suggestions API call description                       |
| Split Out Suggested KWs       | SplitOut             | Split keyword suggestions array          | KW Suggestions API Request   | Set KW Suggestion Fields             |                                                               |
| Set KW Suggestion Fields      | Set                  | Map keyword suggestion fields            | Split Out Suggested KWs      | Add KW Suggestions to Master table   |                                                               |
| Add KW Suggestions to Master table| Airtable          | Insert keyword suggestions into Airtable| Set KW Suggestion Fields     | -                                   | Add to Master All KW Variations table                          |
| KW Ideas API Request          | HTTP Request         | Request keyword ideas                    | Set Fields for API Request   | Split Out KW Ideas                  | Keyword Ideas API call description                             |
| Split Out KW Ideas            | SplitOut             | Split keyword ideas array                | KW Ideas API Request         | Set Keyword Ideas Fields             |                                                               |
| Set Keyword Ideas Fields      | Set                  | Map keyword ideas fields                 | Split Out KW Ideas           | Add KW Ideas to Master table         |                                                               |
| Add KW Ideas to Master table  | Airtable             | Insert keyword ideas into Airtable       | Set Keyword Ideas Fields     | -                                   | Add to Master All KW Variations table                          |
| Autocomplete API Request      | HTTP Request         | Request autocomplete keywords            | Set Fields for API Request   | Split Out Autocomplete               | Autocomplete API call description                             |
| Split Out Autocomplete        | SplitOut             | Split autocomplete keywords array        | Autocomplete API Request     | Set Autocomplete Fields              |                                                               |
| Set Autocomplete Fields       | Set                  | Map autocomplete keyword fields          | Split Out Autocomplete       | Add Autocomplete to Master table     |                                                               |
| Add Autocomplete to Master table| Airtable           | Insert autocomplete keywords into Airtable| Set Autocomplete Fields      | -                                   | Add to Master All KW Variations table                          |
| Serp API Request1             | HTTP Request         | Request Google SERP organic and PAA data| Set Fields for API Request   | Split Out SERP                     | SERP API call description with depth hardcoded                |
| Split Out SERP                | SplitOut             | Split SERP results array                  | Serp API Request1            | Filter SERPs, Filter PAA             |                                                               |
| Filter SERPs                 | Filter               | Filter organic SERP items                 | Split Out SERP               | Set SERP Fields                     |                                                               |
| Set SERP Fields               | Set                  | Map SERP organic result fields            | Filter SERPs                 | Create SERPS                      |                                                               |
| Create SERPS                  | Airtable             | Insert organic SERP results into Airtable| Set SERP Fields              | -                                   | Add to SERP Results table                                     |
| Filter PAA                   | Filter               | Filter People Also Ask items               | Split Out SERP               | Split Out People Also Ask           |                                                               |
| Split Out People Also Ask     | SplitOut             | Split People Also Ask array                | Filter PAA                   | Set PAA Fileds                     |                                                               |
| Set PAA Fileds                | Set                  | Map People Also Ask fields                 | Split Out People Also Ask    | Add PAA to Master Table             |                                                               |
| Add PAA to Master Table       | Airtable             | Insert People Also Ask into Airtable       | Set PAA Fileds               | -                                   | Add to Master All KW Variations table                          |
| Split Out Subtopics           | SplitOut             | Split subtopics array                      | Generate Subtopics API Request| Set Generate Subtopics Fields       |                                                               |
| Set Generate Subtopics Fields | Set                  | Map subtopics fields                       | Split Out Subtopics          | Add PAA to Master Table1            |                                                               |
| Add PAA to Master Table1      | Airtable             | Insert generated subtopics into Airtable  | Set Generate Subtopics Fields| -                                   | Add to Master All KW Variations table                          |
| Generate Subtopics API Request| HTTP Request         | Request subtopics generation               | Set Fields for API Request   | Split Out Subtopics                 | Subtopics API call description                               |
| Airtable (Update Primary Keywords record) | Airtable | Update trigger status to "KW Research Complete" | Set Fields for API Request | -                                   |                                                               |
| Sticky Notes (various)        | Sticky Note          | Setup guidance, usage instructions, block descriptions | - | - | Setup and usage instructions, API notes, and process explanations |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Webhook Node:**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: Unique identifier (e.g., "20d50a4c-b0cf-4f32-82ba-09ca88f3a699")  
   - Purpose: To receive trigger requests from Airtable automation with a query parameter `recordID`.

2. **Create a Set Node ("Set Airtable Fields"):**  
   - Purpose: Initialize variables for Airtable base ID, record ID, and table IDs.  
   - Assign values:  
     - `airtable_base_id`: Your Airtable base ID (string)  
     - `airtable_record_id`: `{{$json.query.recordID}}` (from webhook)  
     - Various table IDs as hardcoded strings (from Airtable base)  
   - Connect Webhook → Set Airtable Fields.

3. **Create Airtable Node ("Get Primary Keyword"):**  
   - Operation: Get record by ID  
   - Base ID: `{{$json.airtable_base_id}}`  
   - Table ID: Primary Keywords table ID (hardcoded or from set variables)  
   - Record ID: `{{$json.airtable_record_id}}`  
   - Credentials: Airtable Personal Access Token  
   - Connect Set Airtable Fields → Get Primary Keyword.

4. **Create Set Node ("Set Fields for API Request"):**  
   - Assign fields from Airtable record:  
     - `Primary Keyword`: `{{$json['Primary Keyword']}}`  
     - `Location`: `{{$json.Location}}`  
     - `Language`: `{{$json.Language}}`  
     - `Limit`: `{{$json.Limit}}`  
     - `Depth`: `{{$json.Depth}}`  
   - Connect Get Primary Keyword → Set Fields for API Request.

5. **Create HTTP Request Nodes for DataForSEO API calls:**  
   Use HTTP Basic Auth credentials for DataForSEO.

   - **Related API Request:**  
     URL: `https://api.dataforseo.com/v3/dataforseo_labs/google/related_keywords/live`  
     Method: POST  
     Body (JSON):  
     ```json
     [{
       "keyword": "{{$json['Primary Keyword']}}",
       "location_name": "{{$json.Location}}",
       "language_name": "{{$json.Language}}",
       "include_serp_info": true,
       "include_seed_keyword": true,
       "limit": {{$json.Limit}},
       "depth": {{$json.Depth}}
     }]
     ```
     Connect Set Fields for API Request → Related API Request.

   - **KW Suggestions API Request:**  
     URL: `https://api.dataforseo.com/v3/dataforseo_labs/google/keyword_suggestions/live`  
     Method: POST  
     Body (JSON) similar, with `"ignore_synonyms": true` and `"include_seed_keyword": false`.  
     Connect Set Fields for API Request → KW Suggestions API Request.

   - **KW Ideas API Request:**  
     URL: `https://api.dataforseo.com/v3/dataforseo_labs/google/keyword_ideas/live`  
     Method: POST  
     Body (JSON) with `"closely_variants": true`, `"ignore_synonyms": false`.  
     Connect Set Fields for API Request → KW Ideas API Request.

   - **Autocomplete API Request:**  
     URL: `https://api.dataforseo.com/v3/serp/google/autocomplete/live/advanced`  
     Method: POST  
     Body includes `"client": "gws-wiz-serp"`.  
     Connect Set Fields for API Request → Autocomplete API Request.

   - **Serp API Request1:**  
     URL: `https://api.dataforseo.com/v3/serp/google/organic/live/advanced`  
     Method: POST  
     Body includes fixed `"depth": 25`, `"people_also_ask_click_depth": 1`.  
     Connect Set Fields for API Request → Serp API Request1.

   - **Generate Subtopics API Request:**  
     URL: `https://api.dataforseo.com/v3/content_generation/generate_sub_topics/live`  
     Method: POST  
     Body with `"topic": "{{$json['Primary Keyword']}}"`.  
     Connect Set Fields for API Request → Generate Subtopics API Request.

6. **Create Split Out Nodes:**  
   For each API request response, create a Split Out node targeting the appropriate JSON array field for items or subtopics.

   - Related API → Split Out Result Items  
   - KW Suggestions API → Split Out Suggested KWs  
   - KW Ideas API → Split Out KW Ideas  
   - Autocomplete API → Split Out Autocomplete  
   - Serp API → Split Out SERP  
   - Generate Subtopics API → Split Out Subtopics

7. **Create Set Nodes to map fields:**  
   For each Split Out node, create a Set node to map and rename fields:

   - Set KW Related Fields (from Split Out Result Items)  
   - Set KW Suggestion Fields (from Split Out Suggested KWs)  
   - Set Keyword Ideas Fields (from Split Out KW Ideas)  
   - Set Autocomplete Fields (from Split Out Autocomplete)  
   - Filter SERPs (filter type = "organic") → Set SERP Fields  
   - Filter PAA (filter type = "people_also_ask") → Split Out People Also Ask → Set PAA Fileds  
   - Set Generate Subtopics Fields (from Split Out Subtopics)

8. **Create Airtable Create Nodes:**  
   Connect each Set Fields node to an Airtable create node that inserts records into the corresponding Airtable table:

   - Related KWs → Master All KW Variations table  
   - KW Suggestions → Master All KW Variations table  
   - KW Ideas → Master All KW Variations table  
   - Autocomplete → Master All KW Variations table  
   - SERP → SERP Results table  
   - People Also Ask → Master All KW Variations table  
   - Subtopics → Master All KW Variations table

9. **Create Filter Nodes for SERP Data:**  
   After "Split Out SERP", create two filters:

   - Filter SERPs: Keep only items with `type == "organic"`  
   - Filter PAA: Keep only items with `type == "people_also_ask"`

   Connect Filter SERPs → Set SERP Fields → Create SERPS  
   Connect Filter PAA → Split Out People Also Ask → Set PAA Fileds → Add PAA to Master Table

10. **Update Airtable Record to Mark Completion:**  
    - Airtable node with "Update" operation.  
    - Target: Primary Keywords table, record ID from "Set Airtable Fields".  
    - Update: Set "Trigger" field to "KW Research Complete".  
    - Connect from "Set Fields for API Request".

11. **Add Sticky Notes:**  
    - Add sticky notes at appropriate positions to document setup instructions, API call descriptions, and process explanations.

12. **Credentials Setup:**  
    - Airtable: Personal Access Token with read/write access to the used base and tables.  
    - DataForSEO: HTTP Basic Authentication credentials with valid API key and secret.

---

### 5. General Notes & Resources

| Note Content                                                                                                              | Context or Link                                                                                          |
|---------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| This workflow requires copying the Airtable base "KW Research Content Ideation" and setting up Airtable Automations as described. | Setup instructions sticky note in workflow; Airtable base link: https://airtable.com/apphzhR0wI16xjJJs    |
| DataForSEO API documentation for endpoint details and parameters.                                                         | https://dataforseo.com/help-center                                                                       |
| Airtable Automation script to trigger n8n webhook upon "Get Keyword Research" trigger status change in Airtable Primary Keywords table. | Provided in sticky note content within workflow                                                         |
| Recommended default values for testing: Location = "United States", Language = "English", Limit = 100, Depth = 2           | Usage instructions sticky note                                                                           |
| Workflow uses HTTP Basic Authentication for DataForSEO and Airtable Personal Access Token for Airtable API access.          | Credential nodes configuration                                                                            |
| Handle API rate limits and errors by monitoring workflow executions and enabling retry or error handling in n8n as needed. | General best practice                                                                                     |

---

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

---