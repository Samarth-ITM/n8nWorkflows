Monitor LinkedIn Posts & Create AI Content Digests with OpenAI and Airtable

https://n8nworkflows.xyz/workflows/monitor-linkedin-posts---create-ai-content-digests-with-openai-and-airtable-8895


# Monitor LinkedIn Posts & Create AI Content Digests with OpenAI and Airtable

### 1. Workflow Overview

This workflow automates the daily monitoring of LinkedIn posts from a defined community of LinkedIn profiles and generates AI-powered content digests stored in Airtable. It is designed for community managers, content creators, and social media teams aiming to efficiently curate and track LinkedIn activities without manual checks.

The workflow is logically divided into the following blocks:

- **1.1 Schedule & Initialization:** Triggers the workflow daily, sets date variables, and loads the list of LinkedIn profiles from Airtable.
- **1.2 Rate Limiting & API Request:** Implements a randomized delay to avoid hitting LinkedIn API rate limits, then fetches the previous day’s posts per profile.
- **1.3 Data Extraction & Cleaning:** Extracts relevant data fields from the LinkedIn API response and cleans profile identifiers.
- **1.4 Community Verification:** Checks if the profile belongs to the monitored community using an Airtable lookup.
- **1.5 AI Content Processing:** Uses OpenAI (via LangChain nodes) to generate short, structured previews of LinkedIn posts.
- **1.6 Duplicate Check & Storage:** Verifies if the post already exists in Airtable to avoid duplication and creates new records for unique posts.
- **1.7 Loop Control and No-Operation Nodes:** Manage batch processing and control flow for edge cases or empty datasets.

---

### 2. Block-by-Block Analysis

#### 1.1 Schedule & Initialization

**Overview:**  
This block triggers the workflow daily, sets the variable for “yesterday” date, and fetches the list of LinkedIn profiles from Airtable to be monitored.

**Nodes Involved:**  
- Schedule Trigger  
- Set Date Variables  
- Daily Check  
- Loop Over Items  
- No Operation, do nothing2  
- Sticky Note (overview and instructions)

**Node Details:**

- **Schedule Trigger**  
  - *Type*: Schedule Trigger  
  - *Role*: Initiates workflow daily (default 24h interval).  
  - *Config*: Uses default interval with no custom time specified (runs every 24 hours).  
  - *Input*: None  
  - *Output*: Triggers next node.  
  - *Failures*: None expected unless n8n scheduling fails.  

- **Set Date Variables**  
  - *Type*: Set  
  - *Role*: Calculates and sets the variable `yesterday` as ISO date string formatted to YYYY-MM-DD.  
  - *Config*: Uses JavaScript expression to subtract 24 hours from current time.  
  - *Key Expression*: `={{ new Date(Date.now() - 24 * 60 * 60 * 1000).toISOString().split('T')[0] }}`  
  - *Input*: Trigger data from Schedule Trigger  
  - *Output*: Adds `yesterday` date field for queries.  
  - *Failures*: Expression errors if runtime date functions fail (unlikely).  

- **Daily Check (Airtable)**  
  - *Type*: Airtable node  
  - *Role*: Fetches the list of LinkedIn profiles (including profile URN) to monitor.  
  - *Config*: Uses Airtable API with credentials; performs search operation with sorting by “Name”.  
  - *Input*: Date variables from previous node.  
  - *Output*: List of profiles for processing.  
  - *Failures*: Airtable API key or base/table misconfiguration, network issues, empty data.  

- **Loop Over Items (Split In Batches)**  
  - *Type*: SplitInBatches  
  - *Role*: Manages batch processing of profiles to respect API quotas and rate limits.  
  - *Config*: Default options; processes profiles one by one or in batches.  
  - *Input*: Profiles list from Airtable node.  
  - *Output*: Passes each profile individually downstream.  
  - *Failures*: None expected, but batch size misconfiguration could cause delays or overload.  

- **No Operation, do nothing2**  
  - *Type*: NoOp  
  - *Role*: Placeholder or pass-through node for flow control when no action needed.  
  - *Input/Output*: Passes input unchanged.  
  - *Failures*: None.

---

#### 1.2 Rate Limiting & API Request

**Overview:**  
This block creates a randomized delay to prevent hitting LinkedIn API rate limits, then queries the LinkedIn Professional Network Data API for posts from the previous day.

**Nodes Involved:**  
- Random Delay (Code)  
- Wait  
- Get Daily LinkedIn Post (HTTP Request)  
- If (conditional to check for posts)  
- No Operation, do nothing

**Node Details:**

- **Random Delay**  
  - *Type*: Code  
  - *Role*: Generates a random delay in seconds between 0.1 and 60 seconds to stagger API calls.  
  - *Config*: Uses JavaScript function generating a float with 1 decimal place.  
  - *Output*: Object with seconds, formatted string, and milliseconds for Wait node.  
  - *Failures*: Possible expression errors if JS code is malformed.  

- **Wait**  
  - *Type*: Wait  
  - *Role*: Pauses execution for the randomized number of seconds generated.  
  - *Config*: Amount set dynamically from Random Delay node output: `={{ $('Random Delay').item.json.seconds }}`.  
  - *Failures*: Timeout or cancellation if workflow interrupted.  

- **Get Daily LinkedIn Post (HTTP Request)**  
  - *Type*: HTTP Request  
  - *Role*: Calls LinkedIn API via RapidAPI to fetch posts for the given LinkedIn profile URN and date (yesterday).  
  - *Config*: URL dynamically constructed using profile URN and query parameter `postedAt` set to `yesterday`.  
  - *Key Expression*: URL template:  
    `https://professional-network-data.p.rapidapi.com/get-profile-posts?start=0&username={{ $('Daily Check').item.json['LinkedIn profile URN'] }}`  
  - *Input*: Profile URN from Airtable  
  - *Output*: API response containing posts data.  
  - *Failures*: API auth errors, rate limit exceeded, network failures, empty or malformed response.  

- **If (Check for Data Presence)**  
  - *Type*: If  
  - *Role*: Checks if the API response contains non-empty `data`.  
  - *Condition*: `$json.data && $json.data.length > 0` (boolean true)  
  - *Output*: Routes to data extraction or no-operation node accordingly.  
  - *Failures*: Expression evaluation errors if JSON data is missing or malstructured.  

- **No Operation, do nothing**  
  - *Type*: NoOp  
  - *Role*: Handles cases where no posts are found; prevents workflow breakage.  
  - *Input/Output*: Pass-through.  

---

#### 1.3 Data Extraction & Cleaning

**Overview:**  
This block extracts structured data from the LinkedIn API response and cleans profile identifiers for further processing.

**Nodes Involved:**  
- Extract Info (Code)  
- Clean Profile Data1 (Set)  
- Lookup CB Community (Airtable)  
- If2 (conditional)  
- Formatting (Set)  
- No Operation, do nothing3  

**Node Details:**

- **Extract Info**  
  - *Type*: Code  
  - *Role*: Parses LinkedIn API JSON response, extracting key fields for each post: profile URLs, URNs, author names, post URLs, timestamps, reactions, comments, content, and a computed date 14 days ahead.  
  - *Key Expressions*: Uses JavaScript to loop over `data` array and construct an array of cleaned post objects with fields like `linkedinProfileUrl`, `postContent`, `twoWeeksFromPublishDate` etc.  
  - *Input*: API response JSON.  
  - *Output*: Array of structured post data objects.  
  - *Failures*: JS execution errors, missing or unexpected fields in API data.  

- **Clean Profile Data1**  
  - *Type*: Set  
  - *Role*: Cleans the LinkedIn profile URN by removing the prefix `urn:li:fsd_profile:` to prepare for community lookup.  
  - *Key Expression*: `={{ $('Extract Info').item.json.linkedinProfileUrn.replace('urn:li:fsd_profile:', '') }}`  
  - *Input*: Extracted post data.  
  - *Output*: Adds `linkedin_profile_clean_urn` field.  
  - *Failures*: Expression failures if field missing or string replacement fails.  

- **Lookup CB Community (Airtable)**  
  - *Type*: Airtable  
  - *Role*: Searches community Airtable base to verify if the cleaned profile URN exists in the community table.  
  - *Config*: Uses Airtable API with filter formula (not fully specified in JSON but logically filtering by `linkedin_profile_clean_urn`).  
  - *Input*: Cleaned profile URN.  
  - *Output*: Community match data or empty.  
  - *Failures*: Airtable connectivity, incorrect filter formula, no match found.  

- **If2**  
  - *Type*: If  
  - *Role*: Checks if the community lookup returned any data (profile is part of community).  
  - *Condition*: Not empty object check on community lookup JSON.  
  - *Output*: Routes to Formatting node if profile is verified community member, else no-op node.  
  - *Failures*: Expression errors or empty data handling.  

- **Formatting**  
  - *Type*: Set  
  - *Role*: Prepares data fields for AI processing and storage, setting standardized fields such as `postcontent`, `authorfullname`, `authorprofileurl`, `authorprofileurn`, `posturl`, and `postdatecreated`.  
  - *Key Expressions*: Mixes values from Extract Info and community lookup outputs.  
  - *Input*: Community verified post data.  
  - *Output*: Structured and formatted data for AI node.  
  - *Failures*: Expression errors if any referenced field missing.  

- **No Operation, do nothing3**  
  - *Type*: NoOp  
  - *Role*: Handles the case where the profile is not in the community; halts further processing for those records.  
  - *Input/Output*: Pass-through.  

---

#### 1.4 AI Content Processing

**Overview:**  
This block uses OpenAI via LangChain nodes to analyze the LinkedIn post content and produce a concise JSON-formatted preview.

**Nodes Involved:**  
- OpenAI Chat Model (LangChain)  
- Structured Output Parser (LangChain)  
- LinkedIn Digestion (LangChain Chain LLM)  
- Lookup Post (Airtable)  
- Check if post exist? (If)  
- No Operation, do nothing1  
- Create Digestion (Airtable)  

**Node Details:**

- **OpenAI Chat Model**  
  - *Type*: LangChain LLM Chat model  
  - *Role*: Sends post content to OpenAI GPT-4o-mini model for content summarization.  
  - *Config*: Model specified as `gpt-4o-mini` with no additional options.  
  - *Credentials*: OpenAI API key configured.  
  - *Input*: Post content from Formatting node.  
  - *Output*: Raw AI response to be parsed.  
  - *Failures*: API key invalid, rate limits, model unavailability, malformed prompt.  

- **Structured Output Parser**  
  - *Type*: LangChain Output Parser  
  - *Role*: Parses the AI response ensuring output is valid JSON with the expected schema containing `post_preview`.  
  - *Config*: Example JSON schema requires a single key: `"post_preview"`.  
  - *Input*: Raw AI text response.  
  - *Output*: Parsed JSON object with preview.  
  - *Failures*: Parsing errors if AI returns invalid JSON or deviates from format.  

- **LinkedIn Digestion (Chain LLM)**  
  - *Type*: LangChain Chain LLM  
  - *Role*: Orchestrates the AI content analysis with prompt template requiring the first 30 characters of the post plus ellipsis in JSON format.  
  - *Config*: Text prompt instructs strict JSON response containing `"post_preview"`.  
  - *Input*: Formatted post content.  
  - *Output*: JSON preview used downstream.  
  - *Failures*: AI or parsing failures as above.  

- **Lookup Post (Airtable)**  
  - *Type*: Airtable  
  - *Role*: Searches Airtable posts table to check if the current post already exists to prevent duplicates.  
  - *Config*: Uses filter formula based on post URN or URL (specific formula not fully detailed).  
  - *Input*: Post metadata including post URL or URN.  
  - *Output*: Existing post data or empty.  
  - *Failures*: Airtable API errors, incorrect filter formula, connectivity issues.  

- **Check if post exist? (If)**  
  - *Type*: If  
  - *Role*: Checks if the lookup returned any existing post record.  
  - *Condition*: Not empty JSON object check.  
  - *Output*: Routes to no-op (if post exists) or to create new record (if not).  
  - *Failures*: Expression errors, handling empty data.  

- **No Operation, do nothing1**  
  - *Type*: NoOp  
  - *Role*: Pass-through when post already exists, skipping creation.  
  - *Input/Output*: Pass-through.  

- **Create Digestion (Airtable)**  
  - *Type*: Airtable  
  - *Role*: Creates a new record in Airtable posts table with all extracted and AI-generated data fields for new posts.  
  - *Config*: Mapping all relevant fields: author info, post content, AI preview, timestamps, engagement metrics.  
  - *Credentials*: Airtable API key configured.  
  - *Input*: Post data and AI summary.  
  - *Output*: New record creation confirmation.  
  - *Failures*: API errors, field mapping errors, data format issues.  

---

#### 1.5 Loop Control and No-Operation Nodes

**Overview:**  
These nodes manage flow control between batches, handle empty sets, and serve as placeholders to maintain workflow integrity.

**Nodes Involved:**  
- No Operation, do nothing  
- No Operation, do nothing1  
- No Operation, do nothing2  
- No Operation, do nothing3  
- Loop Over Items  

**Node Details:**

- **No Operation, do nothing / No Operation, do nothing1 / No Operation, do nothing2 / No Operation, do nothing3**  
  - *Type*: NoOp  
  - *Role*: Pass-through nodes to handle different flow branches where no processing is required.  
  - *Input/Output*: Simple pass-through.  
  - *Failures*: None expected.  

- **Loop Over Items**  
  - *Type*: SplitInBatches  
  - *Role*: Re-initializes or controls batch processing cycles, allowing nodes to process items in manageable groups.  
  - *Input/Output*: Manages batch flow cycles.  
  - *Failures*: Batch size or flow misconfiguration.  

---

### 3. Summary Table

| Node Name                | Node Type                    | Functional Role                               | Input Node(s)                 | Output Node(s)                   | Sticky Note                                                                                              |
|--------------------------|------------------------------|-----------------------------------------------|------------------------------|---------------------------------|---------------------------------------------------------------------------------------------------------|
| Schedule Trigger         | Schedule Trigger             | Triggers workflow daily                        | None                         | Set Date Variables              |                                                                                                         |
| Set Date Variables       | Set                          | Sets "yesterday" date variable                 | Schedule Trigger             | Daily Check                    |                                                                                                         |
| Daily Check              | Airtable                     | Fetches LinkedIn profiles to monitor           | Set Date Variables           | Loop Over Items                | ## 1. Load Community Profiles [Read more about the Airtable node](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.airtable/) Fetches your list of LinkedIn profiles from Airtable to monitor. Each profile should include the LinkedIn profile URL and URN for API calls. |
| Loop Over Items          | SplitInBatches               | Processes profiles in batches                   | Daily Check                  | No Operation, do nothing2; Random Delay |                                                                                                         |
| No Operation, do nothing2 | NoOp                        | Pass-through for no action                      | Loop Over Items              | (none)                        |                                                                                                         |
| Random Delay             | Code                         | Generates random delay for rate limiting       | Loop Over Items              | Wait                          |                                                                                                         |
| Wait                     | Wait                         | Waits random seconds before API call            | Random Delay                 | Get Daily LinkedIn Post        |                                                                                                         |
| Get Daily LinkedIn Post  | HTTP Request                 | Calls LinkedIn API for posts on "yesterday"    | Wait                        | If                           | ## 2. Fetch Recent Posts [Read more about the HTTP Request node](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.httprequest/) Calls the LinkedIn API to get posts from yesterday for each profile. Includes rate limiting to stay within API quotas. |
| If                       | If                           | Checks if API returned posts                    | Get Daily LinkedIn Post      | Extract Info; No Operation, do nothing |                                                                                                         |
| No Operation, do nothing | NoOp                         | Pass-through when no posts returned            | If                          | Loop Over Items               |                                                                                                         |
| Extract Info             | Code                         | Extracts post and author data from API response | If                         | Clean Profile Data1           | ## 3. Extract & Process Data [Read more about the Code node](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.code/) Extracts key information from LinkedIn posts including content, engagement metrics, timestamps, and author details into structured data. |
| Clean Profile Data1      | Set                          | Cleans profile URN for lookup                   | Extract Info                | Lookup CB Community           |                                                                                                         |
| Lookup CB Community      | Airtable                     | Checks if profile is part of community          | Clean Profile Data1          | If2                         |                                                                                                         |
| If2                      | If                           | Routes based on community membership            | Lookup CB Community          | Formatting; No Operation, do nothing3 |                                                                                                         |
| No Operation, do nothing3 | NoOp                        | Pass-through for non-community profiles         | If2                         | Loop Over Items               |                                                                                                         |
| Formatting               | Set                          | Formats data for AI processing                   | If2                         | OpenAI Chat Model             |                                                                                                         |
| OpenAI Chat Model        | LangChain LLM Chat Model     | AI generates preview of LinkedIn post content   | Formatting                  | LinkedIn Digestion            | ## 4. AI Content Analysis [Read more about the LangChain nodes](https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.chainllm/) Uses OpenAI to create concise previews of post content, making it easier to quickly scan and decide which posts deserve attention. |
| Structured Output Parser | LangChain Output Parser      | Parses AI JSON response                          | OpenAI Chat Model           | LinkedIn Digestion            |                                                                                                         |
| LinkedIn Digestion       | LangChain Chain LLM          | Orchestrates AI content digestion and formatting | Structured Output Parser    | Lookup Post                  |                                                                                                         |
| Lookup Post              | Airtable                     | Checks if post already exists in Airtable        | LinkedIn Digestion          | Check if post exist?          |                                                                                                         |
| Check if post exist?     | If                           | Routes based on existence of post record         | Lookup Post                 | No Operation, do nothing1; Create Digestion |                                                                                                         |
| No Operation, do nothing1 | NoOp                        | Skips creation if post exists                     | Check if post exist?         | Loop Over Items               |                                                                                                         |
| Create Digestion         | Airtable                     | Creates new Airtable record for new post          | Check if post exist?         | Loop Over Items               | ## 5. Store Processed Posts [Read more about the Airtable node](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.airtable/) Checks for existing posts to prevent duplicates, then saves new processed posts to Airtable with all extracted data and AI summaries. |
| No Operation, do nothing | NoOp                         | General pass-through nodes                        | Multiple                    | Multiple                     |                                                                                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node:**  
   - Set to run every 24 hours (default interval).  

2. **Add a Set node ("Set Date Variables"):**  
   - Create a string field `yesterday` with value:  
     `={{ new Date(Date.now() - 24 * 60 * 60 * 1000).toISOString().split('T')[0] }}`  

3. **Add an Airtable node ("Daily Check"):**  
   - Operation: Search  
   - Base and Table: Your Airtable base and table containing LinkedIn profiles  
   - Sort by: Name (ascending)  
   - Credentials: Configure Airtable API key  
   - This node fetches the LinkedIn profiles to monitor.  

4. **Add a SplitInBatches node ("Loop Over Items"):**  
   - Default batch size (1 or more depending on API rate limits)  
   - Connect "Daily Check" output to this node input.  

5. **Add a Code node ("Random Delay"):**  
   - JavaScript code:  
     ```js
     function generateRandomSeconds(min = 0.1, max = 60, decimalPlaces = 1) {
       const randomValue = Math.random() * (max - min) + min;
       return Number(randomValue.toFixed(decimalPlaces));
     }
     const seconds = generateRandomSeconds(0.1, 60, 1);
     return [{ seconds: seconds, formatted: `${seconds} seconds`, milliseconds: seconds * 1000 }];
     ```  
   - Connect "Loop Over Items" (batch output) to this node.  

6. **Add a Wait node ("Wait"):**  
   - Set amount: `={{ $('Random Delay').item.json.seconds }}` seconds  
   - Connect "Random Delay" output to "Wait".  

7. **Add an HTTP Request node ("Get Daily LinkedIn Post"):**  
   - Method: GET  
   - URL:  
     `https://professional-network-data.p.rapidapi.com/get-profile-posts?start=0&username={{ $("Loop Over Items").item.json["LinkedIn profile URN"] }}`  
   - Query Parameter: `postedAt` = `={{ $("Set Date Variables").item.json.yesterday }}`  
   - Authentication & Headers: Configure RapidAPI credentials for LinkedIn API  
   - Connect "Wait" output to this node.  

8. **Add an If node ("If"):**  
   - Condition:  
     - Check if `$json.data` exists and has length > 0  
   - Connect "Get Daily LinkedIn Post" output to "If".  

9. **Add a Code node ("Extract Info"):**  
   - JavaScript code to extract and transform LinkedIn post data (as per original code)  
   - Connect "If" true branch to this node.  

10. **Add a Set node ("Clean Profile Data1"):**  
    - Add field `linkedin_profile_clean_urn` with value:  
      `={{ $('Extract Info').item.json.linkedinProfileUrn.replace('urn:li:fsd_profile:', '') }}`  
    - Connect "Extract Info" output to this node.  

11. **Add an Airtable node ("Lookup CB Community"):**  
    - Operation: Search  
    - Base and Table: Airtable community profiles table  
    - Filter formula: Filter by `linkedin_profile_clean_urn` field to check presence in community  
    - Credentials: Airtable API key  
    - Connect "Clean Profile Data1" output to this node.  

12. **Add an If node ("If2"):**  
    - Condition: Check if lookup returned a non-empty object (community member exists)  
    - Connect "Lookup CB Community" output to this node.  

13. **Add a Set node ("Formatting"):**  
    - Assign fields:  
      - `postcontent` = `={{ $('Extract Info').item.json.postContent }}`  
      - `authorfullname` = `={{ $json.Name }}` (from community lookup)  
      - `authorprofileurl` = `={{ $json['LinkedIn profile URL'] }}`  
      - `authorprofileurn` = `={{ $json['LinkedIn profile URN'] }}`  
      - `posturl` = `={{ $('Extract Info').item.json.postUrl }}`  
      - `postdatecreated` = `={{ $('Extract Info').item.json.dateCreated }}`  
    - Connect "If2" true branch to this node.  

14. **Add an OpenAI Chat Model node ("OpenAI Chat Model"):**  
    - Model: `gpt-4o-mini`  
    - Credentials: Configure OpenAI API key  
    - Connect "Formatting" output to this node.  

15. **Add a Structured Output Parser node ("Structured Output Parser"):**  
    - JSON schema example:  
      ```json
      {
        "post_preview": "My top 5 fire tools to help im..."
      }
      ```  
    - Connect "OpenAI Chat Model" output to this node.  

16. **Add a LangChain Chain LLM node ("LinkedIn Digestion"):**  
    - Prompt: Specify that only JSON with `post_preview` containing first 30 chars + "..." must be returned  
    - Connect "Structured Output Parser" output to this node.  

17. **Add an Airtable node ("Lookup Post"):**  
    - Operation: Search  
    - Base and Table: Airtable posts table  
    - Filter formula: To check if post URL or URN already exists  
    - Credentials: Airtable API key  
    - Connect "LinkedIn Digestion" output to this node.  

18. **Add an If node ("Check if post exist?"):**  
    - Condition: Check if lookup returns non-empty result  
    - Connect "Lookup Post" output to this node.  

19. **Add a No Operation node ("No Operation, do nothing1"):**  
    - Connect "Check if post exist?" true branch here (post already exists).  

20. **Add an Airtable node ("Create Digestion"):**  
    - Operation: Create record  
    - Base and Table: Airtable posts table  
    - Map all relevant fields including AI preview, post content, author info, timestamps, engagement data  
    - Credentials: Airtable API key  
    - Connect "Check if post exist?" false branch here (new post).  

21. **Connect "No Operation, do nothing1" and "Create Digestion" outputs to "Loop Over Items" input:**  
    - This loops back for batch processing of next profiles/posts.  

22. **Add No Operation nodes ("No Operation, do nothing" and "No Operation, do nothing3"):**  
    - Connect these to handle empty data branches in the flow accordingly.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 | Context or Link                                                                                   |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| ## 🔍 LinkedIn Post Monitoring & Content Digestion - Automatically monitor LinkedIn posts from your community members and create digestible summaries for content curation. Designed for community managers, content creators, and social media teams. Includes scheduling, API calls, AI summarization, duplicate prevention, and Airtable storage. Join the [Discord](https://discord.com/invite/XPKeKXeB7d) or ask in the [Forum](https://community.n8n.io/) for help.                                                                                                                                                                                                                                                            | Overview sticky note in workflow                                                                 |
| ## 1. Load Community Profiles - [Read more about the Airtable node](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.airtable/) Fetches your list of LinkedIn profiles from Airtable to monitor. Each profile should include the LinkedIn profile URL and URN.                                                                                                                                                                                                                                                                                                                                                                                                                                                                          | Sticky Note on loading profiles                                                                 |
| ## 2. Fetch Recent Posts - [Read more about the HTTP Request node](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.httprequest/) Calls the LinkedIn API to get posts from yesterday for each profile. Includes rate limiting to stay within API quotas.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   | Sticky Note on LinkedIn API fetch                                                                 |
| ## 3. Extract & Process Data - [Read more about the Code node](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.code/) Extracts key information from LinkedIn posts including content, engagement metrics, timestamps, and author details into structured data.                                                                                                                                                                                                                                                                                                                                                                                                                                                                       | Sticky Note on data extraction                                                                     |
| ## 4. AI Content Analysis - [Read more about the LangChain nodes](https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.chainllm/) Uses OpenAI to create concise previews of post content, making it easier to quickly scan and decide which posts deserve attention.                                                                                                                                                                                                                                                                                                                                                                                                                                                    | Sticky Note on AI processing                                                                       |
| ## 5. Store Processed Posts - [Read more about the Airtable node](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.airtable/) Checks for existing posts to prevent duplicates, then saves new processed posts to Airtable with all extracted data and AI summaries.                                                                                                                                                                                                                                                                                                                                                                                                                                                                        | Sticky Note on Airtable storage                                                                    |
| ### ⚠️ Setup Requirements! This workflow requires API credentials and Airtable setup: 1) LinkedIn Professional Network Data API from RapidAPI, 2) OpenAI API key, 3) Airtable base with two tables (profiles and posts), 4) Test with small data set initially.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           | Sticky note with setup instructions                                                              |

---

**Disclaimer:**  
The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data processed is legal and public.