Comprehensive Contact Enrichment with Apollo, LinkedIn, and GPT-4o for HubSpot

https://n8nworkflows.xyz/workflows/comprehensive-contact-enrichment-with-apollo--linkedin--and-gpt-4o-for-hubspot-6103


# Comprehensive Contact Enrichment with Apollo, LinkedIn, and GPT-4o for HubSpot

### 1. Workflow Overview

This workflow, titled **"Comprehensive Contact Enrichment with Apollo, LinkedIn, and GPT-4o for HubSpot"**, is designed to enrich contact records with comprehensive and clean data sourced from Apollo's people matching API, fresh LinkedIn posts, and advanced AI summarization (GPT-4o). The enriched data is then formatted and updated in HubSpot CRM.

**Target Use Cases:**  
- Automatically enhancing CRM contact profiles with verified professional details  
- Gathering recent LinkedIn public activities for contacts  
- Summarizing and cleaning data via AI to ensure CRM-ready structured records  
- Avoiding duplicates and ensuring data quality before syncing with HubSpot

**Logical Blocks:**

- **1.1 Input Reception & Deduplication:** Receives contact input from another workflow and removes duplicates or incomplete emails.  
- **1.2 Apollo Data Enrichment:** Calls Apollo API to fetch detailed contact data based on email.  
- **1.3 LinkedIn Recent Posts Retrieval:** Fetches recent LinkedIn public posts related to the contact using a LinkedIn data API, filters and limits to relevant posts.  
- **1.4 Data Merging:** Combines Apollo enrichment data with recent LinkedIn activity data.  
- **1.5 AI Processing & Summarization:** Uses GPT-4o-powered agents to clean, summarize, and structure the enriched data into a CRM-ready JSON format.  
- **1.6 HubSpot Update:** Updates or enriches the HubSpot contact record with the structured data.

---

### 2. Block-by-Block Analysis

---

#### 1.1 Input Reception & Deduplication

**Overview:**  
Receives input contacts (name and email) from another workflow, normalizes emails, removes duplicates and empty emails for clean processing.

**Nodes Involved:**  
- When Executed by Another Workflow  
- clean (Code Node)  

**Node Details:**

- **When Executed by Another Workflow**  
  - *Type:* Execute Workflow Trigger  
  - *Role:* Entry point that receives JSON input with `name` and `email` fields.  
  - *Configuration:* Receives workflow inputs `name` and `email`.  
  - *Inputs:* External workflow trigger.  
  - *Outputs:* Passes input contacts to next node.  
  - *Edge Cases:* Missing or malformed email may be passed but cleaned downstream.  
  - *Version:* 1.1

- **clean**  
  - *Type:* Code (JavaScript)  
  - *Role:* Deduplicates contacts by normalizing emails (lowercase, trimmed), removes empty or duplicate emails.  
  - *Configuration:* Uses custom JS script that normalizes email, uses a Set for uniqueness, and outputs only unique, valid emails with associated names.  
  - *Key Logic:* Supports nested JSON structure by checking for `output` key fallback.  
  - *Input:* List of contacts from the trigger node.  
  - *Output:* Array of unique contacts with valid emails.  
  - *Edge Cases:* Contacts without emails or duplicate emails are dropped silently.  
  - *Version:* 2

---

#### 1.2 Apollo Data Enrichment

**Overview:**  
For each clean contact email, queries Apollo API to fetch detailed person enrichment data while protecting privacy by disabling personal email and phone reveal.

**Nodes Involved:**  
- Enrich with Apollo  
- Found? (If Node)  

**Node Details:**

- **Enrich with Apollo**  
  - *Type:* HTTP Request  
  - *Role:* Sends POST request to Apollo's people match API endpoint using email query parameter.  
  - *Configuration:*  
    - URL: `https://api.apollo.io/api/v1/people/match`  
    - Method: POST  
    - Query Parameters: `reveal_personal_emails=false`, `reveal_phone_number=false`, `email` from current item.  
    - Headers: API key in `x-api-key` header, plus standard headers (`accept`, `Cache-Control`).  
  - *Input:* Contact email from previous node.  
  - *Output:* Apollo response with potential person enrichment data.  
  - *Edge Cases:* API key invalid or missing, network timeout, no matching person found (empty response).  
  - *Version:* 4.2

- **Found?**  
  - *Type:* If Node  
  - *Role:* Checks if Apollo returned valid person data containing `name` or `email` fields.  
  - *Configuration:* Condition checks existence of `$json.person && ($json.person.name || $json.person.email)`  
  - *Input:* Output from Apollo enrichment.  
  - *Output:*  
    - True branch: continues with data merge and LinkedIn enrichment.  
    - False branch: discards or skips contact (no further processing).  
  - *Edge Cases:* Apollo returns incomplete or no person data.  
  - *Version:* 2.2

---

#### 1.3 LinkedIn Recent Posts Retrieval

**Overview:**  
Retrieves recent LinkedIn public posts for the person via a RapidAPI endpoint, then filters, sorts, limits, and aggregates posts for summarization.

**Nodes Involved:**  
- Loop Over Items  
- Get recent posts  
- Split Out  
- Filter out reshares and old posts  
- Sort by post date  
- Limit to 3  
- Aggregate  

**Node Details:**

- **Loop Over Items**  
  - *Type:* SplitInBatches  
  - *Role:* Processes input contacts in batches to control API usage and concurrency.  
  - *Input:* Contacts that passed Apollo enrichment.  
  - *Output:* Each contact batch sent to LinkedIn post retrieval.  
  - *Version:* 3

- **Get recent posts**  
  - *Type:* HTTP Request  
  - *Role:* Calls RapidAPI LinkedIn profile posts API with `linkedin_url` from Apollo person data.  
  - *Configuration:*  
    - URL: `https://fresh-linkedin-profile-data.p.rapidapi.com/get-profile-posts`  
    - Query: `linkedin_url` and `type=posts`  
    - Headers: `x-rapidapi-host` and `x-rapidapi-key` (user must supply)  
  - *Edge Cases:* API key missing, LinkedIn URL missing or invalid, no posts returned.  
  - *Version:* 4.2

- **Split Out**  
  - *Type:* SplitOut  
  - *Role:* Extracts array of posts from the API response in the `data` field, outputting each post as separate item.  
  - *Version:* 1

- **Filter out reshares and old posts**  
  - *Type:* Filter  
  - *Role:* Removes posts that are reshares (`reshared == true`) or older than 30 days relative to execution date.  
  - *Configuration:*  
    - Conditions: `reshared == false` AND `posted` datetime after or equals 30 days ago.  
  - *Version:* 2.2

- **Sort by post date**  
  - *Type:* Sort  
  - *Role:* Sorts posts in descending order by `posted` date.  
  - *Version:* 1

- **Limit to 3**  
  - *Type:* Limit  
  - *Role:* Limits output to maximum 3 posts to reduce data volume.  
  - *Version:* 1

- **Aggregate**  
  - *Type:* Aggregate  
  - *Role:* Combines the filtered and limited posts back into a single item for further processing.  
  - *Version:* 1

---

#### 1.4 Data Merging

**Overview:**  
Merges Apollo enrichment data and LinkedIn recent posts data into a single combined item for AI summarization.

**Nodes Involved:**  
- Merge  
- Extract fields  

**Node Details:**

- **Merge**  
  - *Type:* Merge  
  - *Role:* Combines two input streams by position: Apollo person data and LinkedIn posts data.  
  - *Configuration:*  
    - Mode: Combine  
    - Include Unpaired: true (to handle missing LinkedIn data)  
  - *Input:*  
    - Input 1: Output from Apollo enrichment branch  
    - Input 2: Output from LinkedIn posts aggregation  
  - *Output:* Combined JSON containing both data sets.  
  - *Version:* 3.1

- **Extract fields**  
  - *Type:* Set  
  - *Role:* Creates two distinct JSON properties: `person_data` and `recent_activity` for AI input.  
  - *Configuration:*  
    - Assignments:  
      - `person_data` = `{{$json.person}}` (Apollo data)  
      - `recent_activity` = `{{$json.data}}` (LinkedIn posts)  
  - *Version:* 3.4

---

#### 1.5 AI Processing & Summarization

**Overview:**  
Uses GPT-4o-based language model chain to convert raw enrichment and LinkedIn activity data into a clean, structured JSON object formatted for HubSpot CRM ingestion.

**Nodes Involved:**  
- Enrichment summary agent  
- OpenAI Chat Model6  
- Auto-fixing Output Parser2  
- Structured Output Parser2  

**Node Details:**

- **Enrichment summary agent**  
  - *Type:* LangChain LLM Chain  
  - *Role:* Defines prompt and logic for AI to generate clean, concise enrichment summaries according to strict rules (job title matching email domain, experience summary, education, recent activity, null handling).  
  - *Configuration:* Prompt includes detailed instructions and example JSON output.  
  - *Input:* `person_data` and `recent_activity` fields extracted earlier.  
  - *Output:* AI-generated structured JSON with keys: email, linkedin_url, job_title, city, state, country_region, experience_summary, education_summary, recent_linkedin_activity.  
  - *Version:* 1.6

- **OpenAI Chat Model6**  
  - *Type:* LangChain OpenAI Chat Model  
  - *Role:* Runs the GPT-4o-mini model to generate AI output based on the prompt.  
  - *Configuration:* Model set to `gpt-4o-mini`, default options.  
  - *Credentials:* OpenAI API key (configured).  
  - *Version:* 1.2

- **Auto-fixing Output Parser2**  
  - *Type:* LangChain Output Parser Autofixing  
  - *Role:* Automatically corrects minor output JSON formatting errors to ensure valid JSON from AI output.  
  - *Version:* 1

- **Structured Output Parser2**  
  - *Type:* LangChain Structured Output Parser  
  - *Role:* Validates AI output against example JSON schema to ensure correct fields and data types.  
  - *Example Schema:* JSON object with enrichment keys and sample values as shown in prompt.  
  - *Version:* 1.2

---

#### 1.6 HubSpot Update

**Overview:**  
Updates or enriches the contact record in HubSpot CRM with the clean, AI-processed enrichment data.

**Nodes Involved:**  
- Enrich in HubSpot1  

**Node Details:**

- **Enrich in HubSpot1**  
  - *Type:* HubSpot node  
  - *Role:* Updates contact by email with enriched fields including city, country, job title, LinkedIn URL, experience, education, last enrichment date, and recent LinkedIn posts.  
  - *Configuration:*  
    - Email from AI output email field  
    - Custom properties mapped to AI output fields  
    - Last enrichment date stored as current date truncated to day (timestamp in milliseconds)  
    - Authenticated via OAuth2 credentials configured externally.  
  - *Edge Cases:* HubSpot API errors, missing email, permission issues.  
  - *Version:* 2.1

---

### 3. Summary Table

| Node Name                     | Node Type                             | Functional Role                                   | Input Node(s)                       | Output Node(s)                   | Sticky Note                                    |
|-------------------------------|-------------------------------------|-------------------------------------------------|-----------------------------------|---------------------------------|------------------------------------------------|
| When Executed by Another Workflow | Execute Workflow Trigger             | Receives input contacts from external workflow  | -                                 | clean                           |                                                |
| clean                         | Code                                | Deduplicates and normalizes input contacts       | When Executed by Another Workflow | Enrich with Apollo              |                                                |
| Enrich with Apollo            | HTTP Request                        | Calls Apollo API for enrichment data             | clean                             | Found?                         | ## Enrich with Apollo                          |
| Found?                       | If                                  | Checks if Apollo returned valid person data      | Enrich with Apollo                | Merge (true branch), Loop Over Items (true branch) |                                                |
| Loop Over Items              | SplitInBatches                      | Processes contacts in batches                     | Found?                           | Merge (second input), Get recent posts (false branch) |                                                |
| Get recent posts             | HTTP Request                       | Fetches recent LinkedIn posts                     | Loop Over Items                  | Split Out                      | ## Grab Recent LinkedIn Posts                  |
| Split Out                   | SplitOut                           | Splits LinkedIn posts array into separate items  | Get recent posts                 | Filter out reshares and old posts |                                                |
| Filter out reshares and old posts | Filter                             | Removes reshares and posts older than 30 days   | Split Out                       | Sort by post date              |                                                |
| Sort by post date           | Sort                               | Sorts posts descending by date                    | Filter out reshares and old posts | Limit to 3                    | Decide how many posts you want to analyse     |
| Limit to 3                  | Limit                              | Limits post count to 3                            | Sort by post date               | Aggregate                     |                                                |
| Aggregate                   | Aggregate                          | Aggregates filtered posts into single item       | Limit to 3                     | Loop Over Items (false branch) |                                                |
| Merge                      | Merge                              | Combines Apollo and LinkedIn data                 | Found? (true), Loop Over Items   | Extract fields                |                                                |
| Extract fields             | Set                                | Prepares `person_data` and `recent_activity` for AI | Merge                          | Enrichment summary agent      |                                                |
| Enrichment summary agent   | LangChain Chain                    | AI prompt logic to summarize and clean data      | Extract fields                  | Enrich in HubSpot1            | ## Summarize enrichment data                   |
| OpenAI Chat Model6         | LangChain OpenAI Chat Model         | GPT-4o-mini model generating AI output           | Enrichment summary agent (AI model input) | Auto-fixing Output Parser2    |                                                |
| Auto-fixing Output Parser2 | LangChain Output Parser Autofixing  | Fixes AI output JSON format errors                | OpenAI Chat Model6 (AI output)  | Enrichment summary agent (AI output) |                                                |
| Structured Output Parser2  | LangChain Structured Output Parser  | Validates AI output JSON against schema          | Auto-fixing Output Parser2 (AI output) | Auto-fixing Output Parser2 (AI input) |                                                |
| Enrich in HubSpot1         | HubSpot Node                      | Updates contact record in HubSpot with enriched data | Enrichment summary agent        | -                             | ## Enrich HubSpot                              |
| Sticky Note                | Sticky Note                       | Comment on Apollo enrichment                      | -                               | -                             | ## Enrich with Apollo                          |
| Sticky Note2               | Sticky Note                       | Comment on AI summarization block                 | -                               | -                             | ## Summarize enrichment data                   |
| Sticky Note3               | Sticky Note                       | Comment on HubSpot enrichment block               | -                               | -                             | ## Enrich HubSpot                              |
| Sticky Note5               | Sticky Note                       | Comment on LinkedIn posts retrieval                | -                               | -                             | ## Grab Recent LinkedIn Posts                  |
| Sticky Note1               | Sticky Note                       | Comment on post limit decision                      | -                               | -                             | Decide how many posts you want to analyse     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Trigger Node**  
   - Add **Execute Workflow Trigger** node named `When Executed by Another Workflow`.  
   - Configure to accept inputs: `name` (string), `email` (string).

2. **Add Deduplication Code Node**  
   - Add a **Code** node named `clean`.  
   - Paste JS code that normalizes emails (trim, lowercase), drops empty/duplicate emails, and outputs unique contacts.  
   - Connect `When Executed by Another Workflow` → `clean`.

3. **Add Apollo Enrichment HTTP Request Node**  
   - Add **HTTP Request** node named `Enrich with Apollo`.  
   - Set URL to `https://api.apollo.io/api/v1/people/match`.  
   - Method: POST.  
   - Query parameters:  
     - `email`: Expression `{{$json.email}}`  
     - `reveal_personal_emails`: `false`  
     - `reveal_phone_number`: `false`  
   - Headers:  
     - `x-api-key`: your Apollo API key  
     - `accept`: `application/json`  
     - `Cache-Control`: `no-cache`  
   - Connect `clean` → `Enrich with Apollo`.

4. **Add Conditional Node to Check Apollo Response**  
   - Add **If** node named `Found?`.  
   - Condition: Check if `$json.person` exists and has `name` or `email`.  
   - Connect `Enrich with Apollo` → `Found?`.

5. **Add Loop for Batch Processing**  
   - Add **SplitInBatches** node named `Loop Over Items`.  
   - Connect `Found?` true branch → `Loop Over Items`.  
   - Connect `Found?` false branch left unconnected or to end.

6. **Add LinkedIn Recent Posts HTTP Request**  
   - Add **HTTP Request** node named `Get recent posts`.  
   - URL: `https://fresh-linkedin-profile-data.p.rapidapi.com/get-profile-posts`.  
   - Query parameters:  
     - `linkedin_url`: Expression `{{$json.person.linkedin_url}}`  
     - `type`: `posts`  
   - Headers:  
     - `x-rapidapi-host`: `fresh-linkedin-profile-data.p.rapidapi.com`  
     - `x-rapidapi-key`: your RapidAPI key  
   - Connect `Loop Over Items` false branch → `Get recent posts`.

7. **Add Split Out Node**  
   - Add **SplitOut** node named `Split Out`.  
   - Field to split out: `data`.  
   - Connect `Get recent posts` → `Split Out`.

8. **Add Filter Node**  
   - Add **Filter** node named `Filter out reshares and old posts`.  
   - Conditions:  
     - `reshared` is `false`  
     - `posted` datetime is after or equal to 30 days ago (`{{$now.minus({ days: 30 }).toISO()}}`)  
   - Connect `Split Out` → `Filter out reshares and old posts`.

9. **Add Sort Node**  
   - Add **Sort** node named `Sort by post date`.  
   - Sort descending by `posted` field.  
   - Connect `Filter out reshares and old posts` → `Sort by post date`.

10. **Add Limit Node**  
    - Add **Limit** node named `Limit to 3`.  
    - Max items: 3.  
    - Connect `Sort by post date` → `Limit to 3`.

11. **Add Aggregate Node**  
    - Add **Aggregate** node named `Aggregate`.  
    - Aggregate all item data.  
    - Connect `Limit to 3` → `Aggregate`.

12. **Merge Apollo and LinkedIn Data**  
    - Add **Merge** node named `Merge`.  
    - Mode: Combine by position, include unpaired.  
    - Connect `Found?` true branch → Merge input 1.  
    - Connect `Aggregate` → Merge input 2.

13. **Add Set Node to Extract Fields**  
    - Add **Set** node named `Extract fields`.  
    - Assign two fields:  
      - `person_data` = `{{$json.person}}`  
      - `recent_activity` = `{{$json.data}}`  
    - Connect `Merge` → `Extract fields`.

14. **Add LangChain LLM Chain Node**  
    - Add **LangChain Chain** node named `Enrichment summary agent`.  
    - Configure prompt with detailed instructions to produce structured JSON with keys: email, linkedin_url, job_title, city, state, country_region, experience_summary, education_summary, recent_linkedin_activity, following the strict rules and formatting.  
    - Connect `Extract fields` → `Enrichment summary agent`.

15. **Add OpenAI Chat Model Node**  
    - Add **LangChain OpenAI Chat Model** node named `OpenAI Chat Model6`.  
    - Model: `gpt-4o-mini`.  
    - Connect `Enrichment summary agent` AI languageModel input → `OpenAI Chat Model6`.

16. **Add Output Parser Nodes**  
    - Add **LangChain Output Parser Autofixing** node named `Auto-fixing Output Parser2`.  
    - Connect `OpenAI Chat Model6` AI output → `Auto-fixing Output Parser2`.  
    - Add **LangChain Structured Output Parser** node named `Structured Output Parser2`.  
    - Configure with example JSON schema matching expected output.  
    - Connect `Auto-fixing Output Parser2` AI output → `Structured Output Parser2`.  
    - Connect `Structured Output Parser2` AI output → `Auto-fixing Output Parser2` AI input (feedback loop).  
    - Connect `Auto-fixing Output Parser2` AI output → `Enrichment summary agent` AI output.

17. **Add HubSpot Contact Update Node**  
    - Add **HubSpot** node named `Enrich in HubSpot1`.  
    - Authentication: OAuth2 (configure credentials).  
    - Contact email: `{{$json.output.email}}` (from AI output).  
    - Map additional fields from AI output to HubSpot properties: city, country_region, job_title, linkedin_account, experience_summary, education_summary, last_enrichment_date (current day timestamp), recent_linkedin_posts.  
    - Connect `Enrichment summary agent` → `Enrich in HubSpot1`.

18. **Activate and Test**  
    - Configure credentials for Apollo API key, RapidAPI key, OpenAI API key, and HubSpot OAuth2.  
    - Test with sample inputs via the trigger node.  
    - Monitor execution for errors, especially API limits and data availability.

---

### 5. General Notes & Resources

| Note Content                                                                                                                        | Context or Link                                                                                                 |
|-------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------|
| Apollo API documentation: https://www.apollo.io/api/docs/                                                                           | Reference for Apollo people match API usage and parameters                                                    |
| RapidAPI LinkedIn posts API requires valid API key from https://rapidapi.com/fresh-linkedin-profile-data/api/fresh-linkedin-profile-data | Used to retrieve LinkedIn public posts                                                                         |
| OpenAI GPT-4o-mini model is a specialized OpenAI model variant for structured summarization                                         | Requires OpenAI API key and quota                                                                             |
| HubSpot OAuth2 credentials must be configured to allow contact updates with custom properties                                        | HubSpot developer portal for API scopes and credentials                                                       |
| The AI prompt enforces strict JSON output formatting ensuring CRM compatibility                                                     | Avoids manual data normalization downstream                                                                   |
| Limit LinkedIn posts to 3 for manageable data size and API costs                                                                     | Adjustable via `Limit to 3` node                                                                              |
| Deduplication step ensures no redundant API calls or CRM updates                                                                    | Important for efficiency and quota management                                                                 |

---

This documentation provides a thorough understanding of the workflow's structure, logic, and integration points, enabling reliable reproduction, modification, and troubleshooting.