Scrape B2B leads from Apollo, generate Groq AI emails, and send via Gmail

https://n8nworkflows.xyz/workflows/scrape-b2b-leads-from-apollo--generate-groq-ai-emails--and-send-via-gmail-14141


# Scrape B2B leads from Apollo, generate Groq AI emails, and send via Gmail

# 1. Workflow Overview

This workflow automates a two-stage B2B outreach pipeline:

1. **Lead acquisition and enrichment** from Apollo based on form inputs
2. **AI cold email generation and email sending** using Groq and Gmail, with Google Sheets acting as the central datastore

It is designed for users who want to:
- collect leads matching a job title and location
- avoid duplicate lead storage
- enrich lead records with contact and company details
- generate structured AI cold emails for pending leads
- send those generated emails through Gmail
- maintain outreach status in Google Sheets

The workflow contains **two explicit entry points**:

- **Trigger 1 — Form Trigger:** collects lead search criteria and launches Apollo scraping/enrichment
- **Trigger 2 — Manual Trigger:** fetches leads from Google Sheets, generates AI emails for pending records, then sends emails for records marked as generated

## 1.1 Lead Input and Search Initiation
This block starts with a form where the user provides the target job title, location, and number of leads. That data is passed to Apollo’s search endpoint.

## 1.2 Lead Extraction, Iteration, and Deduplication
Apollo search results are transformed into one item per person, iterated one by one, and checked against existing Apollo IDs stored in Google Sheets to prevent duplicates.

## 1.3 Lead Enrichment and Sheet Storage
Non-duplicate leads are enriched through Apollo bulk match, cleaned and normalized in a Code node, then saved to the Google Sheet with initial outreach status set to `Pending`.

## 1.4 Pending Lead Selection and AI Email Generation
When the manual trigger runs, the workflow fetches leads from the sheet, keeps only pending leads with an email address, loops over them, and sends each into a Groq-powered AI agent that must return structured JSON with subject and body.

## 1.5 Generated Email Retrieval and Gmail Sending
After emails are generated and saved, the workflow fetches rows again, removes duplicates, filters for rows marked `Mail Generated`, loops over them, sends Gmail messages, and updates the outreach status.

## 1.6 Throttling and Rate-Limit Protection
Wait nodes are placed between Apollo enrichment requests and between email operations to reduce API throttling risk and make batch execution more stable.

---

# 2. Block-by-Block Analysis

## 2.1 Block: Lead Input and Apollo Search

### Overview
This block collects lead generation parameters from a form and submits them to Apollo’s people search endpoint. It is the main entry point for populating the lead database.

### Nodes Involved
- Lead Generation Form
- Apollo — Search Leads

### Node Details

#### Lead Generation Form
- **Type and role:** `n8n-nodes-base.formTrigger`  
  Web form trigger that starts the lead scraping branch.
- **Configuration choices:**
  - Form title: `Lead Generation Form`
  - Description: asks user to enter search details
  - Required fields:
    - `Job Title`
    - `Location`
    - `Number of Leads` as a number field
- **Key expressions or variables used:**
  - Outputs form values as JSON fields, later referenced as:
    - `$json['Job Title']`
    - `$json.Location`
    - `$json['Number of Leads']`
- **Input and output connections:**
  - Entry point node
  - Outputs to `Apollo — Search Leads`
- **Version-specific requirements:**
  - Uses `typeVersion 2.3`
- **Edge cases / failure types:**
  - Invalid or empty user input if form validation is bypassed
  - Unrealistic lead counts may produce API or pagination issues
  - Apollo may return no matches for highly restrictive criteria
- **Sub-workflow reference:** None

#### Apollo — Search Leads
- **Type and role:** `n8n-nodes-base.httpRequest`  
  Sends a POST request to Apollo to search for people matching the submitted criteria.
- **Configuration choices:**
  - Method: `POST`
  - URL: `https://api.apollo.io/api/v1/mixed_people/api_search`
  - Body is JSON
  - Uses generic header authentication via stored Apollo API credential
- **Key expressions or variables used:**
  - `person_titles`: `{{ $json['Job Title'] }}`
  - `person_locations`: `{{ $json.Location }}`
  - `per_page`: `{{ $json['Number of Leads'] }}`
  - `page`: `1`
- **Input and output connections:**
  - Input from `Lead Generation Form`
  - Output to `Extract People Data`
- **Version-specific requirements:**
  - Uses `typeVersion 4.3`
- **Edge cases / failure types:**
  - Apollo authentication failure
  - API quota exhaustion or rate limiting
  - Request body values may be malformed if form output names are changed
  - `per_page` is passed as a string; usually accepted, but numeric handling may vary
  - Pagination is fixed to page 1 only, so larger result sets are not fully traversed
- **Sub-workflow reference:** None

---

## 2.2 Block: Result Extraction, Iteration, and Deduplication

### Overview
This block converts Apollo search results into individual lead items, loops through them, reads existing lead IDs from the sheet, and determines whether each result is already stored.

### Nodes Involved
- Extract People Data
- Process Each Lead
- Fetch Existing Lead IDs
- Merge for Dedup Check
- Deduplicate Leads
- Skip Duplicates

### Node Details

#### Extract People Data
- **Type and role:** `n8n-nodes-base.code`  
  Extracts the `people` array from Apollo search response and emits one item per person.
- **Configuration choices:**
  - Reads `people` from the first input item
  - Also reads the submitted `Job Title` from the form trigger to preserve requested designation
- **Key expressions or variables used:**
  - `const people = $input.first().json.people || []`
  - `$('Lead Generation Form').first().json['Job Title']`
- **Input and output connections:**
  - Input from `Apollo — Search Leads`
  - Output to `Process Each Lead`
- **Version-specific requirements:**
  - Uses Code node `typeVersion 2`
- **Edge cases / failure types:**
  - If Apollo returns no `people` field, output becomes an empty array
  - Cross-node reference to `Lead Generation Form` assumes the trigger executed in the same run
- **Sub-workflow reference:** None

#### Process Each Lead
- **Type and role:** `n8n-nodes-base.splitInBatches`  
  Iterates through one lead at a time for deduplication and enrichment.
- **Configuration choices:**
  - Batch processing with reset option: `={{ $json.person }}`
  - Used as a loop controller
- **Key expressions or variables used:**
  - Reset expression based on current person object
- **Input and output connections:**
  - Input from `Extract People Data`
  - Loop branch output to:
    - `Merge for Dedup Check`
    - `Fetch Existing Lead IDs`
  - Continue branch receives from `Save Leads to Sheet` and from false branch of `Skip Duplicates`
- **Version-specific requirements:**
  - Uses `typeVersion 3`
- **Edge cases / failure types:**
  - Misconfigured batch loops can stall or skip items
  - Reset expression is unusual; if item structure changes, iteration behavior may become inconsistent
- **Sub-workflow reference:** None

#### Fetch Existing Lead IDs
- **Type and role:** `n8n-nodes-base.googleSheets`  
  Reads existing rows from the Google Sheet to collect current Apollo IDs.
- **Configuration choices:**
  - Google Sheet document selected by URL placeholder
  - Sheet name is left unresolved in exported JSON and must be configured
- **Key expressions or variables used:**
  - No complex expressions in the node itself
  - Dedup logic later reads `Apollo ID`
- **Input and output connections:**
  - Input from `Process Each Lead`
  - Output to `Merge for Dedup Check` as input 1
- **Version-specific requirements:**
  - Uses Google Sheets node `typeVersion 4.7`
- **Edge cases / failure types:**
  - Missing spreadsheet URL
  - Missing or wrong sheet name
  - OAuth failure
  - Column header mismatch for `Apollo ID`
- **Sub-workflow reference:** None

#### Merge for Dedup Check
- **Type and role:** `n8n-nodes-base.merge`  
  Combines the current lead item with the sheet rows for comparison.
- **Configuration choices:**
  - Default merge behavior; used simply to feed both streams into the deduplication code
- **Key expressions or variables used:** None directly
- **Input and output connections:**
  - Input 0 from `Process Each Lead`
  - Input 1 from `Fetch Existing Lead IDs`
  - Output to `Deduplicate Leads`
- **Version-specific requirements:**
  - Uses `typeVersion 3.2`
- **Edge cases / failure types:**
  - If merge mode defaults are altered, deduplication code may receive unexpected input structure
- **Sub-workflow reference:** None

#### Deduplicate Leads
- **Type and role:** `n8n-nodes-base.code`  
  Checks whether the current Apollo person ID already exists in the sheet.
- **Configuration choices:**
  - Reads all merged items
  - Assumes first item is the new lead and all following items are existing sheet rows
  - Outputs:
    - `id`
    - `name`
    - `designation`
    - `isDuplicate`
- **Key expressions or variables used:**
  - `const allItems = $input.all();`
  - `const newId = allItems[0].json.id;`
  - `const existingIds = sheetRows.map(i => i.json['Apollo ID']).filter(Boolean);`
  - `existingIds.includes(newId)`
- **Input and output connections:**
  - Input from `Merge for Dedup Check`
  - Output to `Skip Duplicates`
- **Version-specific requirements:**
  - Uses Code node `typeVersion 2`
- **Edge cases / failure types:**
  - Strong dependency on merge item ordering
  - If sheet rows do not contain `Apollo ID`, duplicates will not be detected
  - Contains a stray unused template fragment at the end of the script:
    - `{{ $('Deduplicate Leads').first().json.designation }}`
    This may be harmless if ignored by parser in exported formatting, but should be removed when rebuilding
- **Sub-workflow reference:** None

#### Skip Duplicates
- **Type and role:** `n8n-nodes-base.if`  
  Allows only non-duplicate leads to continue toward enrichment.
- **Configuration choices:**
  - Condition: `$json.isDuplicate` equals `false`
- **Key expressions or variables used:**
  - `={{ $json.isDuplicate }}`
- **Input and output connections:**
  - Input from `Deduplicate Leads`
  - True output to `Wait — Apollo Cooldown (2s)`
  - False output back to `Process Each Lead` to continue the loop
- **Version-specific requirements:**
  - Uses `typeVersion 2.3`
- **Edge cases / failure types:**
  - If dedup code emits non-boolean values, strict comparison may fail
- **Sub-workflow reference:** None

---

## 2.3 Block: Apollo Enrichment, Formatting, and Lead Storage

### Overview
This block enriches each unique Apollo lead, cleans the payload, and stores normalized lead data in Google Sheets with `Pending` outreach status.

### Nodes Involved
- Wait — Apollo Cooldown (2s)
- Apollo — Enrich Lead Data
- Format & Clean Lead Data
- Save Leads to Sheet

### Node Details

#### Wait — Apollo Cooldown (2s)
- **Type and role:** `n8n-nodes-base.wait`  
  Inserts a pause before enrichment API calls.
- **Configuration choices:**
  - Wait amount: `2`
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Input from true branch of `Skip Duplicates`
  - Output to `Apollo — Enrich Lead Data`
- **Version-specific requirements:**
  - Uses `typeVersion 1.1`
- **Edge cases / failure types:**
  - Adds latency in large batches
  - Insufficient if Apollo rate limits are very strict
- **Sub-workflow reference:** None

#### Apollo — Enrich Lead Data
- **Type and role:** `n8n-nodes-base.httpRequest`  
  Uses Apollo bulk match to retrieve richer person/company data.
- **Configuration choices:**
  - Method: `POST`
  - URL: `https://api.apollo.io/api/v1/people/bulk_match`
  - Sends one record in `details`
  - `reveal_personal_emails: true`
  - `reveal_phone_number: false`
  - Auth via Apollo header credential
- **Key expressions or variables used:**
  - Uses current lead ID from:
    - `{{ $('Deduplicate Leads').first().json.id }}`
- **Input and output connections:**
  - Input from `Wait — Apollo Cooldown (2s)`
  - Output to `Format & Clean Lead Data`
- **Version-specific requirements:**
  - Uses `typeVersion 4.3`
- **Edge cases / failure types:**
  - Apollo may not return `matches`
  - Cross-node reference to `Deduplicate Leads` relies on current execution context
  - Personal email reveal may require plan permissions
- **Sub-workflow reference:** None

#### Format & Clean Lead Data
- **Type and role:** `n8n-nodes-base.code`  
  Normalizes enriched Apollo results into a clean row schema for Google Sheets.
- **Configuration choices:**
  - Reads `matches` array
  - Builds output fields:
    - `Apollo ID`
    - `Full Name`
    - `Job Title`
    - `Company`
    - `Email Address`
    - `Phone Number`
    - `Profile URL`
    - `Company Website`
    - `Industry Genre`
  - Derives a title-cased industry label
  - Chooses one phone number and prefixes with `'` to preserve formatting in Sheets
  - Converts LinkedIn paths into full URLs
  - Replaces null-like values with empty strings
- **Key expressions or variables used:**
  - Custom helper functions:
    - `titleCase`
    - `getIndustry`
  - `const matches = $input.first().json.matches || []`
- **Input and output connections:**
  - Input from `Apollo — Enrich Lead Data`
  - Output to `Save Leads to Sheet`
- **Version-specific requirements:**
  - Uses Code node `typeVersion 2`
- **Edge cases / failure types:**
  - Missing organization object
  - Missing names, titles, or emails
  - Obfuscated Apollo values
  - Prefixing `'` to phone values can surprise later consumers if they expect clean numeric strings
- **Sub-workflow reference:** None

#### Save Leads to Sheet
- **Type and role:** `n8n-nodes-base.googleSheets`  
  Upserts cleaned lead records into the master sheet.
- **Configuration choices:**
  - Operation: `appendOrUpdate`
  - Sheet name: `Master Sheet`
  - Matching column: `Apollo ID`
  - Mapping explicitly defines target columns
  - Sets `Outreach Status ` to `Pending`
- **Key expressions or variables used:**
  - Maps multiple fields from current JSON, including:
    - `={{ $json['Apollo ID'] }}`
    - `={{ $json['Full Name'] }}`
    - `={{ $json['Email Address'] }}`
    - etc.
- **Input and output connections:**
  - Input from `Format & Clean Lead Data`
  - Output back to `Process Each Lead` to continue loop
- **Version-specific requirements:**
  - Uses Google Sheets node `typeVersion 4.7`
- **Edge cases / failure types:**
  - Sheet column names include trailing spaces in several headers; exact match is required
  - Wrong spreadsheet URL or missing worksheet causes failure
  - OAuth scope/permission issues
- **Sub-workflow reference:** None

---

## 2.4 Block: Manual Trigger, Pending Lead Filtering, and AI Email Generation

### Overview
This block is launched manually. It fetches rows from the sheet, filters for leads that are ready for email generation, runs an AI agent with a Groq model, parses the output into structured JSON, and saves the generated content.

### Nodes Involved
- When clicking Execute Workflow
- Fetch Leads from Sheet
- Filter Pending Leads
- Email Generation Loop
- AI Cold Email Writer
- Groq LLM (Fast AI)
- Parse Email JSON
- Save Email to Sheet
- Wait — Email Cooldown (4s)

### Node Details

#### When clicking Execute Workflow
- **Type and role:** `n8n-nodes-base.manualTrigger`  
  Manual entry point for the email generation/sending branch.
- **Configuration choices:** Default manual trigger
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Entry point node
  - Output to `Fetch Leads from Sheet`
- **Version-specific requirements:**
  - Uses `typeVersion 1`
- **Edge cases / failure types:**
  - Requires manual execution; not suitable for unattended scheduling as-is
- **Sub-workflow reference:** None

#### Fetch Leads from Sheet
- **Type and role:** `n8n-nodes-base.googleSheets`  
  Reads lead rows from the spreadsheet for AI generation.
- **Configuration choices:**
  - Spreadsheet URL placeholder must be replaced
  - Sheet name is unresolved in export and must be configured
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Input from `When clicking Execute Workflow`
  - Output to `Filter Pending Leads`
- **Version-specific requirements:**
  - Uses `typeVersion 4.7`
- **Edge cases / failure types:**
  - Missing sheet selection
  - Wrong column headers will break downstream expressions
- **Sub-workflow reference:** None

#### Filter Pending Leads
- **Type and role:** `n8n-nodes-base.if`  
  Keeps only rows eligible for email generation.
- **Configuration choices:**
  - Condition 1: `Email Address ` is not empty
  - Condition 2: `Outreach Status ` equals `Pending`
  - `onError`: `continueRegularOutput`
- **Key expressions or variables used:**
  - `={{ $('Fetch Leads from Sheet').item.json['Email Address '] }}`
  - `={{ $('Fetch Leads from Sheet').item.json['Outreach Status '] }}`
- **Input and output connections:**
  - Input from `Fetch Leads from Sheet`
  - True output to `Email Generation Loop`
- **Version-specific requirements:**
  - Uses `typeVersion 2.3`
- **Edge cases / failure types:**
  - Trailing spaces in header names are critical
  - Because `continueRegularOutput` is enabled, some expression errors may not halt the branch
- **Sub-workflow reference:** None

#### Email Generation Loop
- **Type and role:** `n8n-nodes-base.splitInBatches`  
  Iterates pending leads for AI generation.
- **Configuration choices:** Default split/loop behavior
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Input from `Filter Pending Leads`
  - Loop output to `AI Cold Email Writer`
  - Completion/secondary branch to `Fetch Leads for Mail`
  - Receives continuation from `Wait — Email Cooldown (4s)`
- **Version-specific requirements:**
  - Uses `typeVersion 3`
- **Edge cases / failure types:**
  - If no items pass the filter, generation is skipped and downstream send phase may still run on existing sheet state
- **Sub-workflow reference:** None

#### AI Cold Email Writer
- **Type and role:** `@n8n/n8n-nodes-langchain.agent`  
  Uses an LLM agent to generate a cold email as strict JSON.
- **Configuration choices:**
  - Prompt is manually defined
  - Prompt frames the model as an expert B2B cold email copywriter
  - Includes company placeholder instructions:
    - company name
    - services
    - UVP
    - website
    - phone
  - Rules:
    - no “I hope this email finds you well”
    - no pricing mention
    - max 4 paragraphs
    - professional, warm, human tone
  - Output rules demand raw valid JSON only:
    - `Email Subject`
    - `Email Body`
  - Output parser is enabled
- **Key expressions or variables used:**
  - Prompt is static in this export; it does **not** visibly inject lead-specific fields despite sticky note intent
- **Input and output connections:**
  - Main input from `Email Generation Loop`
  - AI model input from `Groq LLM (Fast AI)`
  - Output parser input from `Parse Email JSON`
  - Main output to `Save Email to Sheet`
- **Version-specific requirements:**
  - Uses LangChain Agent `typeVersion 3`
  - Requires compatible n8n AI package support
- **Edge cases / failure types:**
  - Model may return invalid JSON despite instructions
  - Prompt currently lacks explicit lead personalization variables
  - Missing company details in prompt will reduce usefulness
  - AI node compatibility depends on installed n8n version and LangChain integration support
- **Sub-workflow reference:** None

#### Groq LLM (Fast AI)
- **Type and role:** `@n8n/n8n-nodes-langchain.lmChatGroq`  
  Provides the chat model used by the AI agent.
- **Configuration choices:**
  - Model: `qwen/qwen3-32b`
  - Uses Groq API credential
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Connected to `AI Cold Email Writer` as language model
- **Version-specific requirements:**
  - Uses `typeVersion 1`
  - Requires Groq credentials and AI-capable n8n version
- **Edge cases / failure types:**
  - Invalid API key
  - Model deprecation or availability changes
  - Rate limit or token limit issues
- **Sub-workflow reference:** None

#### Parse Email JSON
- **Type and role:** `@n8n/n8n-nodes-langchain.outputParserStructured`  
  Enforces a structured JSON output schema for the generated email.
- **Configuration choices:**
  - JSON schema example requires:
    - `Email Subject`
    - `Email Body`
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Connected to `AI Cold Email Writer` as output parser
- **Version-specific requirements:**
  - Uses `typeVersion 1.3`
- **Edge cases / failure types:**
  - If model output is malformed, parsing can fail
  - Schema is example-based, so strictness depends on parser behavior in installed version
- **Sub-workflow reference:** None

#### Save Email to Sheet
- **Type and role:** `n8n-nodes-base.googleSheets`  
  Intended to store generated email content back into the sheet.
- **Configuration choices:**
  - Operation: `appendOrUpdate`
  - Spreadsheet URL placeholder present
  - Sheet name unresolved in export and must be configured manually
- **Key expressions or variables used:**
  - Not fully defined in export; this node is underconfigured
- **Input and output connections:**
  - Input from `AI Cold Email Writer`
  - Output to `Wait — Email Cooldown (4s)`
- **Version-specific requirements:**
  - Uses `typeVersion 4.7`
- **Edge cases / failure types:**
  - As exported, it lacks explicit column mappings and sheet name
  - Without matching column configuration, it may append duplicate rows instead of updating existing ones
  - It also does not visibly set `Outreach Status ` to `Mail Generated`, though downstream logic expects that status
- **Sub-workflow reference:** None

#### Wait — Email Cooldown (4s)
- **Type and role:** `n8n-nodes-base.wait`  
  Inserts a pause between AI email generations.
- **Configuration choices:**
  - Wait amount: `4`
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Input from `Save Email to Sheet`
  - Output back to `Email Generation Loop`
- **Version-specific requirements:**
  - Uses `typeVersion 1.1`
- **Edge cases / failure types:**
  - Slows throughput
  - May still be insufficient for some model/provider rate limits
- **Sub-workflow reference:** None

---

## 2.5 Block: Fetch Generated Emails and Send Through Gmail

### Overview
After generation, the workflow reloads sheet rows, filters for leads marked as generated, loops through them, sends the emails using Gmail, updates outreach status, and waits before processing the next item.

### Nodes Involved
- Fetch Leads for Mail
- Remove Duplicates
- Filter1
- Loop Over Items
- Send Cold Email via Gmail
- Update the Outreach Status
- Wait — Email Cooldown (60s)

### Node Details

#### Fetch Leads for Mail
- **Type and role:** `n8n-nodes-base.googleSheets`  
  Reads sheet rows again to identify records ready for sending.
- **Configuration choices:**
  - Spreadsheet URL placeholder present
  - Sheet name unresolved in export and must be configured
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Input from completion branch of `Email Generation Loop`
  - Output to `Remove Duplicates`
- **Version-specific requirements:**
  - Uses `typeVersion 4.7`
- **Edge cases / failure types:**
  - Same Google Sheets risks as above
  - If generation step did not update statuses correctly, send phase may find nothing
- **Sub-workflow reference:** None

#### Remove Duplicates
- **Type and role:** `n8n-nodes-base.removeDuplicates`  
  Deduplicates rows before sending.
- **Configuration choices:** Default options only
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Input from `Fetch Leads for Mail`
  - Output to `Filter1`
- **Version-specific requirements:**
  - Uses `typeVersion 2`
- **Edge cases / failure types:**
  - Since no field selection is shown, deduplication behavior depends on node defaults in the installed n8n version
- **Sub-workflow reference:** None

#### Filter1
- **Type and role:** `n8n-nodes-base.filter`  
  Keeps only rows whose outreach status equals `Mail Generated`.
- **Configuration choices:**
  - Condition on `Outreach Status ` equals `Mail Generated`
- **Key expressions or variables used:**
  - `={{ $json['Outreach Status '] }}`
- **Input and output connections:**
  - Input from `Remove Duplicates`
  - Output to `Loop Over Items`
- **Version-specific requirements:**
  - Uses `typeVersion 2.3`
- **Edge cases / failure types:**
  - If `Save Email to Sheet` never writes `Mail Generated`, this branch will be empty
  - Again depends on exact header name with trailing space
- **Sub-workflow reference:** None

#### Loop Over Items
- **Type and role:** `n8n-nodes-base.splitInBatches`  
  Iterates over generated email rows to send one at a time.
- **Configuration choices:** Default loop behavior
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Input from `Filter1`
  - Loop branch to `Send Cold Email via Gmail`
  - Continuation from `Wait — Email Cooldown (60s)`
- **Version-specific requirements:**
  - Uses `typeVersion 3`
- **Edge cases / failure types:**
  - Empty set if no mail-generated rows exist
- **Sub-workflow reference:** None

#### Send Cold Email via Gmail
- **Type and role:** `n8n-nodes-base.gmail`  
  Sends the generated email to the lead.
- **Configuration choices:**
  - Recipient: `Fetch Leads from Sheet` → `Email Address `
  - Subject: current item field `Email Subject `
  - Message: current item field `Email Body`
  - Sender name: `Your Name`
  - Attribution disabled
  - Email type: plain text
- **Key expressions or variables used:**
  - `={{ $('Fetch Leads from Sheet').item.json['Email Address '] }}`
  - `={{ $json['Email Body'] }}`
  - `={{ $json['Email Subject '] }}`
- **Input and output connections:**
  - Input from `Loop Over Items`
  - Output to `Update the Outreach Status`
- **Version-specific requirements:**
  - Uses Gmail node `typeVersion 2.2`
- **Edge cases / failure types:**
  - Recipient expression references `Fetch Leads from Sheet` instead of the current send loop item; this can send to the wrong address or fail in batch contexts
  - Gmail OAuth permission errors
  - Sending limits or account restrictions
  - Missing subject/body if generation storage failed
- **Sub-workflow reference:** None

#### Update the Outreach Status
- **Type and role:** `n8n-nodes-base.googleSheets`  
  Intended to update the sheet after successful sending.
- **Configuration choices:**
  - Operation: `update`
  - Spreadsheet URL placeholder present
  - Sheet name unresolved in export and must be configured
  - Node has internal notes in flow
- **Key expressions or variables used:** Not shown in export
- **Input and output connections:**
  - Input from `Send Cold Email via Gmail`
  - Output to `Wait — Email Cooldown (60s)`
- **Version-specific requirements:**
  - Uses Google Sheets node `typeVersion 4.2`
- **Edge cases / failure types:**
  - As exported, update keys/column mapping are not visible, so it may be incomplete
  - If not configured with row matching and status field update, sent leads will not be marked correctly
- **Sub-workflow reference:** None

#### Wait — Email Cooldown (60s)
- **Type and role:** `n8n-nodes-base.wait`  
  Adds a substantial delay between outbound emails.
- **Configuration choices:**
  - Wait amount: `60`
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Input from `Update the Outreach Status`
  - Output back to `Loop Over Items`
- **Version-specific requirements:**
  - Uses `typeVersion 1.1`
- **Edge cases / failure types:**
  - Large campaigns will take significant time
  - Helpful for Gmail sender reputation and quota pacing
- **Sub-workflow reference:** None

---

## 2.6 Block: Documentation and In-Canvas Notes

### Overview
These sticky notes document workflow intent, required credentials, and the meaning of major sections. They are not executable but are important for reproduction and maintenance.

### Nodes Involved
- Sticky Note
- Sticky Note1
- Sticky Note2
- Sticky Note3
- Sticky Note4
- Sticky Note5
- Sticky Note6

### Node Details

#### Sticky Note
- **Type and role:** `n8n-nodes-base.stickyNote`  
  High-level operational summary and setup instructions.
- **Configuration choices:**
  - Contains overview of both triggers and required credentials
- **Input and output connections:** None
- **Edge cases / failure types:** None
- **Sub-workflow reference:** None

#### Sticky Note1
- **Type and role:** Sticky note for Trigger 1
- **Configuration choices:** Documents the lead input form section
- **Input and output connections:** None

#### Sticky Note2
- **Type and role:** Sticky note for lead enrichment block
- **Configuration choices:** Documents Apollo enrichment behavior
- **Input and output connections:** None

#### Sticky Note3
- **Type and role:** Sticky note for deduplication block
- **Configuration choices:** Explains duplicate prevention intent
- **Input and output connections:** None

#### Sticky Note4
- **Type and role:** Sticky note for save-leads block
- **Configuration choices:** Explains saving verified leads to the sheet
- **Input and output connections:** None

#### Sticky Note5
- **Type and role:** Sticky note for AI email generation block
- **Configuration choices:** States that Groq generates personalized emails and stores them
- **Input and output connections:** None

#### Sticky Note6
- **Type and role:** Sticky note for Trigger 2
- **Configuration choices:** Explains manual generation and sending phase
- **Input and output connections:** None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Lead Generation Form | n8n-nodes-base.formTrigger | Collect lead search criteria and start Apollo branch |  | Apollo — Search Leads | ## Trigger 1 —Lead Input Form<br>User submits job title, location and number of leads required. This triggers the workflow automatically. |
| Apollo — Search Leads | n8n-nodes-base.httpRequest | Search Apollo for matching people | Lead Generation Form | Extract People Data | ## Lead Discovery and Enrichment<br>Apollo searches for matching leads based on form input. Each lead is enriched with email, phone number, LinkedIn URL and company information. |
| Extract People Data | n8n-nodes-base.code | Convert Apollo people array into one item per lead | Apollo — Search Leads | Process Each Lead | ## Lead Discovery and Enrichment<br>Apollo searches for matching leads based on form input. Each lead is enriched with email, phone number, LinkedIn URL and company information. |
| Process Each Lead | n8n-nodes-base.splitInBatches | Loop through each lead candidate | Extract People Data, Save Leads to Sheet, Skip Duplicates | Merge for Dedup Check, Fetch Existing Lead IDs | ## Lead Discovery and Enrichment<br>Apollo searches for matching leads based on form input. Each lead is enriched with email, phone number, LinkedIn URL and company information. |
| Fetch Existing Lead IDs | n8n-nodes-base.googleSheets | Read existing Apollo IDs from the sheet | Process Each Lead | Merge for Dedup Check | ## Duplicate Prevention<br>Each lead is checked against existing records in the Google Sheet. Duplicate leads are automatically skipped to avoid redundant outreach. |
| Merge for Dedup Check | n8n-nodes-base.merge | Combine current lead with existing rows for deduplication | Process Each Lead, Fetch Existing Lead IDs | Deduplicate Leads | ## Duplicate Prevention<br>Each lead is checked against existing records in the Google Sheet. Duplicate leads are automatically skipped to avoid redundant outreach. |
| Deduplicate Leads | n8n-nodes-base.code | Determine whether current Apollo ID already exists | Merge for Dedup Check | Skip Duplicates | ## Duplicate Prevention<br>Each lead is checked against existing records in the Google Sheet. Duplicate leads are automatically skipped to avoid redundant outreach. |
| Skip Duplicates | n8n-nodes-base.if | Route duplicate vs non-duplicate leads | Deduplicate Leads | Wait — Apollo Cooldown (2s), Process Each Lead | ## Duplicate Prevention<br>Each lead is checked against existing records in the Google Sheet. Duplicate leads are automatically skipped to avoid redundant outreach. |
| Wait — Apollo Cooldown (2s) | n8n-nodes-base.wait | Throttle Apollo enrichment requests | Skip Duplicates | Apollo — Enrich Lead Data | ## Lead Discovery and Enrichment<br>Apollo searches for matching leads based on form input. Each lead is enriched with email, phone number, LinkedIn URL and company information. |
| Apollo — Enrich Lead Data | n8n-nodes-base.httpRequest | Enrich one Apollo lead with detailed contact/company data | Wait — Apollo Cooldown (2s) | Format & Clean Lead Data | ## Lead Discovery and Enrichment<br>Apollo searches for matching leads based on form input. Each lead is enriched with email, phone number, LinkedIn URL and company information. |
| Format & Clean Lead Data | n8n-nodes-base.code | Normalize enriched lead fields for sheet storage | Apollo — Enrich Lead Data | Save Leads to Sheet | ## Save Leads<br>All enriched and verified leads are saved to the Google Sheet with outreach status set to Pending. |
| Save Leads to Sheet | n8n-nodes-base.googleSheets | Upsert lead into master sheet with Pending status | Format & Clean Lead Data | Process Each Lead | ## Save Leads<br>All enriched and verified leads are saved to the Google Sheet with outreach status set to Pending. |
| When clicking Execute Workflow | n8n-nodes-base.manualTrigger | Manual start for email generation/sending branch |  | Fetch Leads from Sheet | ## Trigger 2 — Email Generator and Sender<br>Run manually when ready to generate and send emails. Generates AI cold emails for all Pending leads then sends to all Mail Generated leads via Gmail. |
| Fetch Leads from Sheet | n8n-nodes-base.googleSheets | Read leads for AI generation | When clicking Execute Workflow | Filter Pending Leads | ## Trigger 2 — Email Generator and Sender<br>Run manually when ready to generate and send emails. Generates AI cold emails for all Pending leads then sends to all Mail Generated leads via Gmail. |
| Filter Pending Leads | n8n-nodes-base.if | Keep only pending rows with email address | Fetch Leads from Sheet | Email Generation Loop | ## Trigger 2 — Email Generator and Sender<br>Run manually when ready to generate and send emails. Generates AI cold emails for all Pending leads then sends to all Mail Generated leads via Gmail. |
| Email Generation Loop | n8n-nodes-base.splitInBatches | Loop through pending leads for AI email generation | Filter Pending Leads, Wait — Email Cooldown (4s) | Fetch Leads for Mail, AI Cold Email Writer | ## AI Email Generation<br>Groq LLM generates a personalized cold email for each lead based on their job title, company and industry. Subject and body are saved back to the sheet. |
| AI Cold Email Writer | @n8n/n8n-nodes-langchain.agent | Generate structured cold email JSON | Email Generation Loop, Groq LLM (Fast AI), Parse Email JSON | Save Email to Sheet | ## AI Email Generation<br>Groq LLM generates a personalized cold email for each lead based on their job title, company and industry. Subject and body are saved back to the sheet. |
| Groq LLM (Fast AI) | @n8n/n8n-nodes-langchain.lmChatGroq | Provide Groq chat model for AI generation |  | AI Cold Email Writer | ## AI Email Generation<br>Groq LLM generates a personalized cold email for each lead based on their job title, company and industry. Subject and body are saved back to the sheet. |
| Parse Email JSON | @n8n/n8n-nodes-langchain.outputParserStructured | Enforce JSON output structure for subject and body |  | AI Cold Email Writer | ## AI Email Generation<br>Groq LLM generates a personalized cold email for each lead based on their job title, company and industry. Subject and body are saved back to the sheet. |
| Save Email to Sheet | n8n-nodes-base.googleSheets | Store generated email content back in sheet | AI Cold Email Writer | Wait — Email Cooldown (4s) | ## AI Email Generation<br>Groq LLM generates a personalized cold email for each lead based on their job title, company and industry. Subject and body are saved back to the sheet. |
| Wait — Email Cooldown (4s) | n8n-nodes-base.wait | Throttle AI email generation loop | Save Email to Sheet | Email Generation Loop | ## AI Email Generation<br>Groq LLM generates a personalized cold email for each lead based on their job title, company and industry. Subject and body are saved back to the sheet. |
| Fetch Leads for Mail | n8n-nodes-base.googleSheets | Read sheet rows for sending phase | Email Generation Loop | Remove Duplicates | ## Trigger 2 — Email Generator and Sender<br>Run manually when ready to generate and send emails. Generates AI cold emails for all Pending leads then sends to all Mail Generated leads via Gmail. |
| Remove Duplicates | n8n-nodes-base.removeDuplicates | Deduplicate rows before sending | Fetch Leads for Mail | Filter1 | ## Trigger 2 — Email Generator and Sender<br>Run manually when ready to generate and send emails. Generates AI cold emails for all Pending leads then sends to all Mail Generated leads via Gmail. |
| Filter1 | n8n-nodes-base.filter | Keep only rows marked Mail Generated | Remove Duplicates | Loop Over Items | ## Trigger 2 — Email Generator and Sender<br>Run manually when ready to generate and send emails. Generates AI cold emails for all Pending leads then sends to all Mail Generated leads via Gmail. |
| Loop Over Items | n8n-nodes-base.splitInBatches | Loop through generated emails for sending | Filter1, Wait — Email Cooldown (60s) | Send Cold Email via Gmail | ## Trigger 2 — Email Generator and Sender<br>Run manually when ready to generate and send emails. Generates AI cold emails for all Pending leads then sends to all Mail Generated leads via Gmail. |
| Send Cold Email via Gmail | n8n-nodes-base.gmail | Send generated email to lead | Loop Over Items | Update the Outreach Status | ## Trigger 2 — Email Generator and Sender<br>Run manually when ready to generate and send emails. Generates AI cold emails for all Pending leads then sends to all Mail Generated leads via Gmail. |
| Update the Outreach Status | n8n-nodes-base.googleSheets | Mark outreach state after send | Send Cold Email via Gmail | Wait — Email Cooldown (60s) | ## Trigger 2 — Email Generator and Sender<br>Run manually when ready to generate and send emails. Generates AI cold emails for all Pending leads then sends to all Mail Generated leads via Gmail. |
| Wait — Email Cooldown (60s) | n8n-nodes-base.wait | Delay between outbound emails | Update the Outreach Status | Loop Over Items | ## Trigger 2 — Email Generator and Sender<br>Run manually when ready to generate and send emails. Generates AI cold emails for all Pending leads then sends to all Mail Generated leads via Gmail. |
| Sticky Note | n8n-nodes-base.stickyNote | Workspace documentation note |  |  | ## Scrape leads from Apollo and send AI-powered cold emails — Full B2B Outreach Automation<br>OVERVIEW<br>Two triggers for complete control over your outreach pipeline.<br>TRIGGER 1 — LEAD SCRAPER (Form Trigger)<br>Fill the form with Job Title, Location and Number of Leads.<br>Apollo finds, enriches and saves leads to Google Sheet automatically.<br>TRIGGER 2 — EMAIL GENERATOR AND SENDER (Manual Trigger)<br>Run manually when ready.<br>Generates AI cold emails for all Pending leads, then sends emails to all Mail Generated leads via Gmail.<br>CREDENTIALS REQUIRED<br>- Apollo API Key (Header Auth)<br>- Google Sheets OAuth2<br>- Groq API Key (At console.groq.com)<br>- Gmail OAuth2<br>SETUP INSTRUCTIONS<br>- Replace YOUR_GOOGLE_SHEET_URL with your Google Sheet URL<br>- Add Apollo API Key in Header Auth credential<br>- Add Groq API Key in Groq credential<br>- Connect your Gmail account<br>- Update the email prompt with your company details |
| Sticky Note1 | n8n-nodes-base.stickyNote | Workspace note for Trigger 1 |  |  | ## Trigger 1 —Lead Input Form<br>User submits job title, location and number of leads required. This triggers the workflow automatically. |
| Sticky Note2 | n8n-nodes-base.stickyNote | Workspace note for enrichment block |  |  | ## Lead Discovery and Enrichment<br>Apollo searches for matching leads based on form input. Each lead is enriched with email, phone number, LinkedIn URL and company information. |
| Sticky Note3 | n8n-nodes-base.stickyNote | Workspace note for duplicate prevention |  |  | ## Duplicate Prevention<br>Each lead is checked against existing records in the Google Sheet. Duplicate leads are automatically skipped to avoid redundant outreach. |
| Sticky Note4 | n8n-nodes-base.stickyNote | Workspace note for lead storage |  |  | ## Save Leads<br>All enriched and verified leads are saved to the Google Sheet with outreach status set to Pending. |
| Sticky Note5 | n8n-nodes-base.stickyNote | Workspace note for AI email generation |  |  | ## AI Email Generation<br>Groq LLM generates a personalized cold email for each lead based on their job title, company and industry. Subject and body are saved back to the sheet. |
| Sticky Note6 | n8n-nodes-base.stickyNote | Workspace note for Trigger 2 |  |  | ## Trigger 2 — Email Generator and Sender<br>Run manually when ready to generate and send emails. Generates AI cold emails for all Pending leads then sends to all Mail Generated leads via Gmail. |

---

# 4. Reproducing the Workflow from Scratch

Below is a practical rebuild sequence.

## Prerequisites
1. Create or prepare a **Google Sheet** with a worksheet named **Master Sheet**.
2. Add these columns exactly, preserving spaces where shown:
   - `Apollo ID`
   - `Full Name `
   - `Job Title`
   - `Company`
   - `Email Address `
   - `Phone Number`
   - `Profile URL `
   - `Company Website`
   - `Industry Genre `
   - `Email Subject `
   - `Email Body`
   - `Outreach Status `
3. Create credentials in n8n:
   - **Apollo API Key** as `HTTP Header Auth`
     - Header name typically `X-Api-Key` or Apollo’s current required header format
   - **Google Sheets OAuth2**
   - **Groq API**
   - **Gmail OAuth2**
4. Confirm your n8n instance supports:
   - Form Trigger
   - LangChain Agent node
   - Groq Chat Model node
   - Structured Output Parser node

## Build steps

1. **Create a Form Trigger node**
   - Name: `Lead Generation Form`
   - Title: `Lead Generation Form`
   - Description: `Enter details to search and generate leads automatically`
   - Add fields:
     - Text: `Job Title` required
     - Text: `Location` required
     - Number: `Number of Leads` required

2. **Add an HTTP Request node**
   - Name: `Apollo — Search Leads`
   - Method: `POST`
   - URL: `https://api.apollo.io/api/v1/mixed_people/api_search`
   - Authentication: Generic credential type → Header Auth → Apollo credential
   - Send body as JSON
   - Body:
     - `person_titles`: array with the submitted job title
     - `person_locations`: array with submitted location
     - `per_page`: submitted number of leads
     - `page`: `1`
   - Connect `Lead Generation Form` → `Apollo — Search Leads`

3. **Add a Code node**
   - Name: `Extract People Data`
   - Purpose: turn the `people` array into one item per person
   - Use logic equivalent to:
     - read `people`
     - read original `Job Title`
     - return items with `id`, `person`, `designation`
   - Connect `Apollo — Search Leads` → `Extract People Data`

4. **Add a Split In Batches node**
   - Name: `Process Each Lead`
   - Use it as the loop controller for scraped leads
   - Connect `Extract People Data` → `Process Each Lead`

5. **Add a Google Sheets node**
   - Name: `Fetch Existing Lead IDs`
   - Operation: read/get rows from your lead sheet
   - Select the spreadsheet and `Master Sheet`
   - Connect `Process Each Lead` → `Fetch Existing Lead IDs`

6. **Add a Merge node**
   - Name: `Merge for Dedup Check`
   - Connect:
     - `Process Each Lead` → Merge input 1
     - `Fetch Existing Lead IDs` → Merge input 2

7. **Add a Code node**
   - Name: `Deduplicate Leads`
   - Implement logic:
     - take the current lead ID from the first merged item
     - collect all existing `Apollo ID` values from sheet rows
     - output `isDuplicate`
   - Remove the stray unused expression fragment from the original export
   - Connect `Merge for Dedup Check` → `Deduplicate Leads`

8. **Add an IF node**
   - Name: `Skip Duplicates`
   - Condition:
     - Boolean `isDuplicate` equals `false`
   - Connect `Deduplicate Leads` → `Skip Duplicates`

9. **Add a Wait node**
   - Name: `Wait — Apollo Cooldown (2s)`
   - Wait amount: `2 seconds`
   - Connect true output of `Skip Duplicates` → `Wait — Apollo Cooldown (2s)`

10. **Connect the false output of `Skip Duplicates` back to `Process Each Lead`**
    - This continues the loop when a duplicate is found

11. **Add an HTTP Request node**
    - Name: `Apollo — Enrich Lead Data`
    - Method: `POST`
    - URL: `https://api.apollo.io/api/v1/people/bulk_match`
    - Authentication: same Apollo header credential
    - JSON body:
      - `details`: array containing current lead `id`
      - `reveal_personal_emails`: `true`
      - `reveal_phone_number`: `false`
    - Prefer referencing the current item directly rather than cross-node references if possible
    - Connect `Wait — Apollo Cooldown (2s)` → `Apollo — Enrich Lead Data`

12. **Add a Code node**
    - Name: `Format & Clean Lead Data`
    - Recreate the normalization logic:
      - parse `matches`
      - derive full name
      - select first deduplicated title
      - choose best phone
      - prefix phone with `'` for Sheets if needed
      - normalize LinkedIn URL
      - derive `Industry Genre`
      - ensure empty strings for missing fields
    - Connect `Apollo — Enrich Lead Data` → `Format & Clean Lead Data`

13. **Add a Google Sheets node**
    - Name: `Save Leads to Sheet`
    - Operation: `Append or Update`
    - Spreadsheet: your Google Sheet
    - Sheet: `Master Sheet`
    - Match column: `Apollo ID`
    - Map fields:
      - `Apollo ID` ← `Apollo ID`
      - `Full Name ` ← `Full Name`
      - `Job Title` ← `Job Title`
      - `Company` ← `Company`
      - `Email Address ` ← `Email Address`
      - `Phone Number` ← `Phone Number`
      - `Profile URL ` ← `Profile URL`
      - `Company Website` ← `Company Website`
      - `Industry Genre ` ← `Industry Genre`
      - `Outreach Status ` ← `Pending`
    - Connect `Format & Clean Lead Data` → `Save Leads to Sheet`

14. **Connect `Save Leads to Sheet` back to `Process Each Lead`**
    - This continues the scraped lead loop

15. **Add a Manual Trigger node**
    - Name: `When clicking Execute Workflow`
    - This starts the email generation/sending branch

16. **Add a Google Sheets node**
    - Name: `Fetch Leads from Sheet`
    - Read rows from `Master Sheet`
    - Connect `When clicking Execute Workflow` → `Fetch Leads from Sheet`

17. **Add an IF node**
    - Name: `Filter Pending Leads`
    - Conditions:
      - `Email Address ` is not empty
      - `Outreach Status ` equals `Pending`
    - Connect `Fetch Leads from Sheet` → `Filter Pending Leads`

18. **Add a Split In Batches node**
    - Name: `Email Generation Loop`
    - Connect true output of `Filter Pending Leads` → `Email Generation Loop`

19. **Add a Groq Chat Model node**
    - Name: `Groq LLM (Fast AI)`
    - Model: `qwen/qwen3-32b`
    - Attach Groq credential

20. **Add a Structured Output Parser node**
    - Name: `Parse Email JSON`
    - Define schema/example with:
      - `Email Subject`
      - `Email Body`

21. **Add an AI Agent node**
    - Name: `AI Cold Email Writer`
    - Prompt type: define manually
    - Enable output parser
    - Connect:
      - `Groq LLM (Fast AI)` as language model
      - `Parse Email JSON` as output parser
      - `Email Generation Loop` main output → `AI Cold Email Writer`
    - Improve the prompt by injecting row data, for example:
      - lead full name
      - company
      - job title
      - industry
    - Also replace the company placeholders with your actual company details

22. **Add a Google Sheets node**
    - Name: `Save Email to Sheet`
    - Operation: `Append or Update`
    - Spreadsheet: same Google Sheet
    - Sheet: `Master Sheet`
    - Match by `Apollo ID` or another stable unique key
    - Map/update:
      - `Email Subject ` ← generated `Email Subject`
      - `Email Body` ← generated `Email Body`
      - `Outreach Status ` ← `Mail Generated`
      - Preserve lead identity fields as needed for update matching
    - Connect `AI Cold Email Writer` → `Save Email to Sheet`

23. **Add a Wait node**
    - Name: `Wait — Email Cooldown (4s)`
    - Set to `4 seconds`
    - Connect `Save Email to Sheet` → `Wait — Email Cooldown (4s)`

24. **Connect `Wait — Email Cooldown (4s)` back to `Email Generation Loop`**
    - This continues the generation loop

25. **Add another Google Sheets node**
    - Name: `Fetch Leads for Mail`
    - Read rows from `Master Sheet`
    - Connect the completion output of `Email Generation Loop` → `Fetch Leads for Mail`

26. **Add a Remove Duplicates node**
    - Name: `Remove Duplicates`
    - Deduplicate on a stable field such as `Apollo ID` or `Email Address `
    - Connect `Fetch Leads for Mail` → `Remove Duplicates`

27. **Add a Filter node**
    - Name: `Filter1`
    - Condition:
      - `Outreach Status ` equals `Mail Generated`
    - Connect `Remove Duplicates` → `Filter1`

28. **Add a Split In Batches node**
    - Name: `Loop Over Items`
    - Connect `Filter1` → `Loop Over Items`

29. **Add a Gmail node**
    - Name: `Send Cold Email via Gmail`
    - Operation: Send email
    - Gmail credential: connect your Gmail account
    - Recommended configuration:
      - To: current item `Email Address `
      - Subject: current item `Email Subject `
      - Message: current item `Email Body`
      - Sender name: your real sender name
      - Email type: text
      - Append attribution: false
    - Important: use the **current loop item**, not a cross-reference to `Fetch Leads from Sheet`
    - Connect `Loop Over Items` → `Send Cold Email via Gmail`

30. **Add a Google Sheets node**
    - Name: `Update the Outreach Status`
    - Operation: `Update`
    - Match row by `Apollo ID`
    - Set `Outreach Status ` to something like `Sent`
    - Optionally store sent timestamp
    - Connect `Send Cold Email via Gmail` → `Update the Outreach Status`

31. **Add a Wait node**
    - Name: `Wait — Email Cooldown (60s)`
    - Set to `60 seconds`
    - Connect `Update the Outreach Status` → `Wait — Email Cooldown (60s)`

32. **Connect `Wait — Email Cooldown (60s)` back to `Loop Over Items`**
    - This continues the send loop

33. **Add sticky notes if desired**
    - Recreate the workspace notes for operations, credentials, and usage instructions

## Recommended fixes while rebuilding
34. **Fix underconfigured Google Sheets nodes**
    - In the export, `Save Email to Sheet`, `Fetch Leads from Sheet`, `Fetch Leads for Mail`, and `Update the Outreach Status` are missing some visible sheet details
    - Explicitly set the spreadsheet and worksheet on all of them

35. **Fix email generation personalization**
    - The current AI prompt does not inject lead-specific variables
    - Include expressions such as:
      - company
      - job title
      - industry
      - full name
    - Otherwise all generated emails may be generic

36. **Fix Gmail recipient expression**
    - Do not use `$('Fetch Leads from Sheet')...`
    - Use the current item field from the send loop

37. **Fix status transitions**
    - After generation: set `Outreach Status ` to `Mail Generated`
    - After send: set `Outreach Status ` to `Sent`

38. **Preserve exact Google Sheets column names**
    - Several fields include trailing spaces
    - If you rename them to cleaner headers, update all node expressions consistently

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Scrape leads from Apollo and send AI-powered cold emails — Full B2B Outreach Automation | Workspace overview note |
| Two triggers for complete control over your outreach pipeline | Workflow design note |
| Trigger 1 — Lead Scraper: Fill the form with Job Title, Location and Number of Leads. Apollo finds, enriches and saves leads to Google Sheet automatically. | Operational note |
| Trigger 2 — Email Generator and Sender: Run manually when ready. Generates AI cold emails for all Pending leads, then sends emails to all Mail Generated leads via Gmail. | Operational note |
| Credentials required: Apollo API Key (Header Auth), Google Sheets OAuth2, Groq API Key, Gmail OAuth2 | Setup note |
| Setup instructions: Replace `YOUR_GOOGLE_SHEET_URL` with your Google Sheet URL; add Apollo API key; add Groq API key; connect Gmail; update the email prompt with your company details | Setup note |
| Trigger 1 — Lead Input Form: User submits job title, location and number of leads required. This triggers the workflow automatically. | Sticky note context |
| Lead Discovery and Enrichment: Apollo searches for matching leads based on form input. Each lead is enriched with email, phone number, LinkedIn URL and company information. | Sticky note context |
| Duplicate Prevention: Each lead is checked against existing records in the Google Sheet. Duplicate leads are automatically skipped to avoid redundant outreach. | Sticky note context |
| Save Leads: All enriched and verified leads are saved to the Google Sheet with outreach status set to Pending. | Sticky note context |
| AI Email Generation: Groq LLM generates a personalized cold email for each lead based on their job title, company and industry. Subject and body are saved back to the sheet. | Sticky note context |
| Trigger 2 — Email Generator and Sender: Run manually when ready to generate and send emails. Generates AI cold emails for all Pending leads then sends to all Mail Generated leads via Gmail. | Sticky note context |

## Final implementation observations
- The workflow logic is solid at a high level, but the exported configuration is **partially incomplete** for several Google Sheets nodes.
- The send phase depends on `Outreach Status ` becoming `Mail Generated`, but that update is not clearly configured in the exported `Save Email to Sheet` node.
- The AI prompt currently uses placeholders for company details and does not clearly inject lead-specific variables, so personalization is weaker than the sticky note suggests.
- The Gmail recipient expression should be corrected to use the current loop item rather than the earlier sheet fetch node.