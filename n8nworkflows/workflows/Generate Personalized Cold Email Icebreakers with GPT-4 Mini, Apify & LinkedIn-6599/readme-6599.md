Generate Personalized Cold Email Icebreakers with GPT-4 Mini, Apify & LinkedIn

https://n8nworkflows.xyz/workflows/generate-personalized-cold-email-icebreakers-with-gpt-4-mini--apify---linkedin-6599


# Generate Personalized Cold Email Icebreakers with GPT-4 Mini, Apify & LinkedIn

### 1. Workflow Overview

This workflow automates the generation of personalized cold email icebreakers targeted at dental industry leads in the US. It is designed to enrich raw lead data by scraping detailed LinkedIn profile information, then uses OpenAI’s GPT-4 Mini model to create concise, tailored email openers. The workflow updates lead records progressively to avoid redundant processing.

Logical blocks grouped by their functional roles and dependencies:

- **1.1 Input Reception & Filtering**: Import raw leads from a Google Sheet and filter those with valid email addresses to target only reachable prospects.
- **1.2 LinkedIn Data Enrichment**: For each filtered lead, retrieve detailed LinkedIn profile data via Apify’s LinkedIn scraping API.
- **1.3 Data Aggregation & Simplification**: Aggregate LinkedIn data from the API response and simplify fields to prepare structured input for AI processing.
- **1.4 AI-Powered Icebreaker Generation**: Use OpenAI GPT-4 Mini to generate personalized, concise cold email icebreakers based on enriched data.
- **1.5 Output Storage & Lead Status Update**: Append generated icebreakers to a Google Sheet and mark leads as “enriched” in the original list to prevent reprocessing.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Filtering

- **Overview:**  
  This block imports raw lead data from a Google Sheet and filters out leads lacking email addresses, ensuring the workflow targets only valid contacts.

- **Nodes Involved:**  
  - When clicking ‘Execute workflow’ (Manual Trigger)  
  - Get Raw Un-enriched Leads (Google Sheets)  
  - hasEmail? (Filter)  

- **Node Details:**

  - **When clicking ‘Execute workflow’**  
    - *Type & Role:* Manual trigger to start workflow execution.  
    - *Configuration:* Default manual trigger without parameters.  
    - *Input/Output:* No input; outputs trigger to next node.  
    - *Edge Cases:* Manual execution required; no automated scheduling.  

  - **Get Raw Un-enriched Leads**  
    - *Type & Role:* Google Sheets node to read leads marked as “un-enriched” from a specific sheet/tab.  
    - *Configuration:* Filters rows where `status` column equals "un-enriched". Uses a connected Google Sheets OAuth2 credential.  
    - *Input/Output:* Triggered by manual start; outputs array of raw leads data.  
    - *Edge Cases:* Requires correct Google Sheets ID and sheet name; fails if credentials or sheet access is invalid.  

  - **hasEmail?**  
    - *Type & Role:* Filter node to pass only leads with non-empty `email` fields forward.  
    - *Configuration:* Condition checks if `email` field is not empty (strict type validation, case sensitive).  
    - *Input/Output:* Input is raw leads; output splits into leads with valid emails only.  
    - *Edge Cases:* Leads without emails are discarded; strict validation may exclude malformed emails.  

#### 2.2 LinkedIn Data Enrichment

- **Overview:**  
  For each lead with an email, this block loops over the items, injects Apify API credentials, invokes Apify’s LinkedIn scraping API, and aggregates the returned data.

- **Nodes Involved:**  
  - Loop Over Items (Split In Batches)  
  - Set Apify Tokens (Set)  
  - Call Apify LinkedIn API (HTTP Request)  
  - Aggregate (Aggregate)  

- **Node Details:**

  - **Loop Over Items**  
    - *Type & Role:* Splits the filtered leads into individual items for sequential processing.  
    - *Configuration:* Default batch processing settings.  
    - *Input/Output:* Input from filter; outputs single lead per iteration to next node.  
    - *Edge Cases:* Large data sets may slow processing; no concurrency control specified.  

  - **Set Apify Tokens**  
    - *Type & Role:* Sets Apify API token and LinkedIn scraper actor ID as variables for request authentication.  
    - *Configuration:* Two string fields: `apifyAPIKey` and `apifyActorID`, expected to be provided by user.  
    - *Input/Output:* Input is single lead; outputs with added credentials for API call.  
    - *Edge Cases:* Workflow fails if tokens are missing or incorrect.  

  - **Call Apify LinkedIn API**  
    - *Type & Role:* Makes POST HTTP request to Apify API to scrape LinkedIn profile data based on the lead’s LinkedIn URL.  
    - *Configuration:*  
      - URL dynamically uses `apifyActorID` and sends JSON body with `profileUrls` array containing current lead’s LinkedIn URL.  
      - Headers include `Authorization: Bearer <apifyAPIKey>`.  
      - Accepts JSON response with profile data.  
    - *Input/Output:* Input is enriched with tokens; outputs scraped LinkedIn data.  
    - *Edge Cases:* API rate limits, invalid URLs, or incorrect tokens cause errors or empty data.  

  - **Aggregate**  
    - *Type & Role:* Aggregates API response data from the Apify call, consolidating it into a single item for further processing.  
    - *Configuration:* Aggregates all item data into one array under the `data` key.  
    - *Input/Output:* Input is API response per item; outputs aggregated data for AI processing.  
    - *Edge Cases:* Empty or malformed API response may cause aggregation issues.  

#### 2.3 Data Aggregation & Simplification

- **Overview:**  
  This block simplifies the aggregated LinkedIn data and combines it with lead metadata for input to the AI node.

- **Nodes Involved:**  
  - Simplify Fields for AI Agent (Set)  

- **Node Details:**

  - **Simplify Fields for AI Agent**  
    - *Type & Role:* Set node formats combined data into concise, usable variables for AI consumption.  
    - *Configuration:* Assigns fields like `firstName`, `lastName`, `companyName`, `headline`, `currentJobDurationInYrs`, `email`, `organizationShortDescription`, `organizationCity`, and `organizationState` from aggregated data and original lead info. Uses expressions to extract data from aggregated JSON and lead items.  
    - *Input/Output:* Input is aggregated LinkedIn data plus lead info; output is simplified JSON with relevant fields.  
    - *Edge Cases:* Missing fields in API response or lead data can result in empty or invalid variables for AI.  

#### 2.4 AI-Powered Icebreaker Generation

- **Overview:**  
  This block sends simplified lead data to an OpenAI GPT-4 Mini model to generate personalized cold email icebreakers in a strict JSON format.

- **Nodes Involved:**  
  - Generate Personalized Icebreaker (OpenAI)  

- **Node Details:**

  - **Generate Personalized Icebreaker**  
    - *Type & Role:* OpenAI node configured to use GPT-4.1-mini model to generate short personalized email openers.  
    - *Configuration:*  
      - System message defines role as a sales assistant specialized in dental industry cold emails.  
      - Instruction to use laconic tone, shorten company/location names, avoid generic compliments, and output a JSON object with an `icebreaker` property.  
      - User message passes simplified lead data and scraped LinkedIn profile details via expressions.  
      - Outputs JSON parsed response for downstream use.  
    - *Input/Output:* Input is simplified lead data; output is AI-generated icebreaker JSON.  
    - *Edge Cases:* AI rate limits, malformed input data, or API errors may cause failures or non-compliant output.  

#### 2.5 Output Storage & Lead Status Update

- **Overview:**  
  This block appends the generated icebreakers alongside lead data into a Google Sheet and updates the original leads sheet marking processed leads as “enriched” to avoid repetition.

- **Nodes Involved:**  
  - Append Enriched Icebreaker (Google Sheets)  
  - Update Un-enriched List (Google Sheets)  

- **Node Details:**

  - **Append Enriched Icebreaker**  
    - *Type & Role:* Google Sheets node appending new rows to a designated sheet with enriched lead data and generated icebreakers.  
    - *Configuration:* Maps columns such as `id`, `name`, `email`, `company`, `headline`, `linkedIn`, and generated `icebreaker` from AI output and lead data. Uses OAuth2 credentials connected to user’s Google account.  
    - *Input/Output:* Input is AI output and original lead data; outputs trigger to update node.  
    - *Edge Cases:* Sheet access or quota errors; data mapping inconsistencies.  

  - **Update Un-enriched List**  
    - *Type & Role:* Updates the original leads Google Sheet entry to change `status` from “un-enriched” to “enriched”.  
    - *Configuration:* Matches rows by `id` and sets `status` to “enriched”. Maintains other lead fields unchanged.  
    - *Input/Output:* Input is lead item with enriched status; outputs trigger back to loop for next lead.  
    - *Edge Cases:* Google Sheets API update errors; incorrect matching column causes data mismatch.  

---

### 3. Summary Table

| Node Name                   | Node Type                | Functional Role                          | Input Node(s)               | Output Node(s)                  | Sticky Note                                                                                                                      |
|-----------------------------|--------------------------|----------------------------------------|-----------------------------|--------------------------------|---------------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger           | Workflow start trigger                  | None                        | Get Raw Un-enriched Leads      |                                                                                                                                 |
| Get Raw Un-enriched Leads    | Google Sheets            | Load raw leads marked as “un-enriched”| When clicking ‘Execute workflow’ | hasEmail?                    | ## Pre-process raw leads - Connect your Google Sheet account with n8n. - Choose the file that contains your downloaded CSV raw lead. - Discard leads that do not have work email |
| hasEmail?                   | Filter                   | Filter leads with valid email addresses| Get Raw Un-enriched Leads    | Loop Over Items                | ## Pre-process raw leads - Connect your Google Sheet account with n8n. - Choose the file that contains your downloaded CSV raw lead. - Discard leads that do not have work email |
| Loop Over Items             | Split In Batches         | Iterate over each lead individually     | hasEmail?                   | Set Apify Tokens               |                                                                                                                                 |
| Set Apify Tokens            | Set                      | Add Apify API credentials               | Loop Over Items             | Call Apify LinkedIn API        | ## Get data from LinkedIn, and clean it. In the "Set Apify Tokens" Node do the following: 1. set the apifyAPIKey with your own Apify API key [(get it from here)](https://console.apify.com/settings/integrations), 2. set the apifyActorID with the Apify Scraper ID of your choice. [This](https://console.apify.com/actors/2SyF0bVxmgGr8IVCZ) actor works best for scraping LinkedIn profile |
| Call Apify LinkedIn API     | HTTP Request             | Scrape LinkedIn profile data via Apify | Set Apify Tokens            | Aggregate                     | ## Get data from LinkedIn, and clean it. In the "Set Apify Tokens" Node do the following: 1. set the apifyAPIKey with your own Apify API key [(get it from here)](https://console.apify.com/settings/integrations), 2. set the apifyActorID with the Apify Scraper ID of your choice. [This](https://console.apify.com/actors/2SyF0bVxmgGr8IVCZ) actor works best for scraping LinkedIn profile |
| Aggregate                  | Aggregate                | Aggregate API response data             | Call Apify LinkedIn API      | Simplify Fields for AI Agent   |                                                                                                                                 |
| Simplify Fields for AI Agent| Set                      | Prepare simplified data for AI          | Aggregate                   | Generate Personalized Icebreaker |                                                                                                                                 |
| Generate Personalized Icebreaker | OpenAI (Langchain)      | Generate personalized cold email icebreakers | Simplify Fields for AI Agent | Append Enriched Icebreaker    | ## Generate personalized icebreaker - Cleans content to feed into AI agent - The AI agent goes through the scraped data and writes hyper-personalized icebreaker for cold email - Change the system prompt in the AI agent node to fit your use case. - OpenAI GPT 4.1 mini works best without breaking your bank. |
| Append Enriched Icebreaker | Google Sheets            | Append icebreaker and lead data to sheet| Generate Personalized Icebreaker | Update Un-enriched List        | ## Append & Update Google Sheet - The "Append Enriched Icebreaker" node appends the icebreaker along with the respective lead info into a new Google Sheet. - The "Update Un-enriched List" node updates the original Google Sheet to mark the processed lead as "enriched" from "un-enriched" |
| Update Un-enriched List    | Google Sheets            | Mark leads as enriched in original list| Append Enriched Icebreaker  | Loop Over Items                | ## Append & Update Google Sheet - The "Append Enriched Icebreaker" node appends the icebreaker along with the respective lead info into a new Google Sheet. - The "Update Un-enriched List" node updates the original Google Sheet to mark the processed lead as "enriched" from "un-enriched" |
| Sticky Note                | Sticky Note              | Documentation and overview note         | None                        | None                         | See detailed note content in sections 5 and node descriptions.                                                                   |
| Sticky Note1               | Sticky Note              | Preprocessing instructions               | None                        | None                         | See detailed note content in sections 5 and node descriptions.                                                                   |
| Sticky Note2               | Sticky Note              | Apify token and actor setup instructions| None                        | None                         | See detailed note content in sections 5 and node descriptions.                                                                   |
| Sticky Note3               | Sticky Note              | AI icebreaker generation instructions   | None                        | None                         | See detailed note content in sections 5 and node descriptions.                                                                   |
| Sticky Note4               | Sticky Note              | Google Sheets append/update instructions| None                        | None                         | See detailed note content in sections 5 and node descriptions.                                                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger Node**  
   - Type: Manual Trigger  
   - Purpose: To start workflow execution manually.  

2. **Add Google Sheets Node to Read Raw Leads**  
   - Type: Google Sheets (Read)  
   - Configure: Connect your Google Sheets OAuth2 credentials.  
   - Document ID: Your Google Sheet ID containing raw lead data.  
   - Sheet Name: Tab containing leads (e.g., “100 Leads”).  
   - Filters: Filter rows where `status` is "un-enriched" to only process new leads.  

3. **Add Filter Node to Check for Email Presence**  
   - Type: Filter  
   - Condition: Field `email` is not empty (string notEmpty).  
   - Connect input from Google Sheets node.  

4. **Add SplitInBatches Node to Loop Over Leads**  
   - Type: SplitInBatches  
   - Input: Connect from Filter node output (leads with email).  
   - Purpose: Process each lead individually to avoid API limits and manage flow.  

5. **Add Set Node to Store Apify API Credentials**  
   - Type: Set  
   - Add two fields:  
     - `apifyAPIKey`: Your Apify API key (string).  
     - `apifyActorID`: Apify LinkedIn Scraper Actor ID to use (string).  
   - Connect from Loop node output.  

6. **Add HTTP Request Node to Call Apify LinkedIn API**  
   - Type: HTTP Request (POST)  
   - URL: `https://api.apify.com/v2/acts/{{ $json.apifyActorID }}/run-sync-get-dataset-items`  
   - Headers:  
     - `Accept`: `application/json`  
     - `Authorization`: `Bearer {{ $json.apifyAPIKey }}`  
   - Body (JSON):  
     ```json
     {
       "profileUrls": ["{{ $json.linkedin_url }}"]
     }
     ```  
   - Connect from Set Apify Tokens node.  

7. **Add Aggregate Node**  
   - Type: Aggregate  
   - Aggregate all incoming items into a single array under `data`.  
   - Connect from HTTP Request node output.  

8. **Add Set Node to Simplify & Prepare AI Input**  
   - Type: Set  
   - Map fields from aggregated data and lead info to simplified variables:  
     - `firstName`, `lastName`, `companyName`, `headline`, `currentJobDurationInYrs`, `email`, `organizationShortDescription`, `organizationCity`, `organizationState`  
   - Use expressions to extract data from `$json.data[0]` and original lead item as needed.  
   - Connect from Aggregate node.  

9. **Add OpenAI Node to Generate Personalized Icebreaker**  
   - Type: OpenAI (Langchain)  
   - Credentials: Connect your OpenAI API key.  
   - Model: Use GPT-4.1-mini.  
   - Messages:  
     - System prompt: Role as intelligent sales assistant targeting dental industry leads, with instructions on tone, format, and variables.  
     - User prompt: Pass simplified fields and LinkedIn data as context.  
   - Enable JSON output parsing.  
   - Connect from Simplify Fields node.  

10. **Add Google Sheets Node to Append Icebreaker Data**  
    - Type: Google Sheets (Append)  
    - Document ID and Sheet Name: Target sheet to store enriched leads and icebreakers.  
    - Map columns: `id`, `name`, `email`, `company`, `headline`, `linkedIn`, `icebreaker` (from AI output).  
    - Credentials: Google Sheets OAuth2.  
    - Connect from OpenAI node.  

11. **Add Google Sheets Node to Update Original Leads Status**  
    - Type: Google Sheets (Update)  
    - Document ID and Sheet Name: Original leads sheet.  
    - Matching column: `id`  
    - Update field: Set `status` to “enriched”.  
    - Connect from Append Enriched Icebreaker node.  

12. **Connect Update Node Back to Loop**  
    - Connect output of Update Un-enriched List node to Loop Over Items node to process next lead until done.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                    | Context or Link                                                   |
|-------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------|
| This workflow automates personalized email icebreaker generation for dental industry leads using Apify LinkedIn scraping and OpenAI GPT-4 Mini.| Workflow purpose and overview note (Sticky Note)                  |
| Use Apify Apollo scraper to obtain raw leads, download CSV, and import into Google Sheets connected to n8n for best results.                   | Lead sourcing recommendation                                     |
| Apify tokens must be set: get API key at https://console.apify.com/settings/integrations and use LinkedIn scraper actor https://console.apify.com/actors/2SyF0bVxmgGr8IVCZ | Apify API setup instructions (Sticky Note2)                      |
| AI prompt is highly customizable depending on niche, offer, and pain points; openAI GPT-4.1 mini balances cost and capability well.              | AI generation instructions (Sticky Note3)                        |
| Google Sheets nodes handle data persistence: one appends enriched data, the other updates original lead status to avoid duplicate processing.    | Google Sheets usage notes (Sticky Note4)                         |
| Working demo video available: https://www.youtube.com/watch?v=Ah4Ynj-56sM                                                                         | Demo video link                                                  |

---

**Disclaimer:** The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This process strictly complies with current content policies and contains no illegal, offensive, or protected material. All data processed is legal and public.