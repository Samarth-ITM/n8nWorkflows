Generate and qualify local business leads using Google Sheets, RapidAPI and OpenAI

https://n8nworkflows.xyz/workflows/generate-and-qualify-local-business-leads-using-google-sheets--rapidapi-and-openai-14366


# Generate and qualify local business leads using Google Sheets, RapidAPI and OpenAI

# 1. Workflow Overview

This workflow automatically generates and qualifies local business leads from spreadsheet-defined search requests. It reads search terms and locations from Google Sheets, queries a RapidAPI endpoint for local business data, formats the returned records into a lead-friendly structure, uses OpenAI to assess lead quality and generate an outreach opener, and finally stores the enriched leads back into a Google Sheets results sheet.

Typical use cases include:

- Building outbound prospecting lists for local service sales
- Enriching location-based business searches with contact details
- Prioritizing leads using AI-based qualification
- Maintaining a lightweight lead database in Google Sheets

## 1.1 Scheduled Input Reception

The workflow starts on a recurring schedule and pulls rows from a Google Sheet that act as search instructions. Each input row is expected to contain at least:

- `Keyword`
- `Location`
- `ID`

These values are later used to build the business search query and track which results came from which search request.

## 1.2 Business Discovery via API

For each search request row, the workflow calls a RapidAPI endpoint that searches local businesses matching the keyword and location. It explicitly asks the API to extract emails and contact details where available.

## 1.3 Result Normalization

The raw API response is transformed into one item per business with standardized fields such as:

- Search ID
- Business Name
- Phone
- Email
- Address
- Website

This block also inserts fallback placeholder values when the API does not return specific fields.

## 1.4 AI-Based Lead Qualification

Each normalized business item is sent to OpenAI. The model is instructed to:

- classify the business category
- summarize the business briefly
- assign a lead score
- estimate confidence
- produce a short outreach opener

The prompt requires JSON-only output.

## 1.5 Result Persistence

The enriched records are written into a Google Sheets results sheet using an append-or-update operation keyed on `Search ID`. The sheet stores both raw business contact data and AI-generated qualification fields.

---

# 2. Block-by-Block Analysis

## 2.1 Scheduled Input Reception

### Overview
This block triggers the workflow automatically and loads search instructions from Google Sheets. It is the entry point of the workflow and defines the cadence and source of all downstream lead generation.

### Nodes Involved
- `Schedule Trigger`
- `Read Search Requests`

### Node Details

#### Schedule Trigger
- **Type and technical role:** `n8n-nodes-base.scheduleTrigger`  
  Entry-point trigger node that runs the workflow on a recurring interval.
- **Configuration choices:**  
  Configured with an interval rule based on `minutes`. The JSON does not specify a numeric value in the visible snippet, so the workflow is set to run on a minute-based recurrence according to the saved schedule configuration.
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - Input: none
  - Output: `Read Search Requests`
- **Version-specific requirements:**  
  Uses `typeVersion: 1.3`.
- **Edge cases or potential failure types:**  
  - Misconfigured schedule causing overly frequent execution
  - Workflow disabled or inactive
  - Unintended duplicate runs if downstream deduplication is not strong enough
- **Sub-workflow reference:**  
  None.

#### Read Search Requests
- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Reads rows from a Google Sheets worksheet that contains search instructions.
- **Configuration choices:**  
  - Uses a Google Sheet by document URL/ID
  - Reads from the worksheet identified as `gid=0`
  - No advanced options are enabled
- **Key expressions or variables used:**  
  Downstream nodes expect this node to provide:
  - `$json.Keyword`
  - `$json.Location`
  - `$json.ID`
- **Input and output connections:**  
  - Input: `Schedule Trigger`
  - Output: `Search Businesses API`
- **Version-specific requirements:**  
  Uses `typeVersion: 4.7`, which requires current Google Sheets node support and valid Google credentials.
- **Edge cases or potential failure types:**  
  - Google OAuth authentication failure
  - Invalid spreadsheet URL or inaccessible sheet
  - Missing expected columns such as `Keyword`, `Location`, or `ID`
  - Empty sheet producing no downstream searches
- **Sub-workflow reference:**  
  None.

---

## 2.2 Business Discovery via API

### Overview
This block converts each spreadsheet row into a location-based search query and sends it to the external business search API. It is responsible for fetching local business records and contact metadata.

### Nodes Involved
- `Search Businesses API`

### Node Details

#### Search Businesses API
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Calls the RapidAPI-backed local business search endpoint.
- **Configuration choices:**  
  - Method is implicitly the default request method for this node configuration
  - URL: `https://local-business-data.p.rapidapi.com/search`
  - Query parameters:
    - `query` = keyword + `' in '` + location
    - `limit` = `25`
    - `extract_emails_and_contacts` = `true`
  - Header sending is enabled, which is important because RapidAPI usually requires authentication headers such as API key and host
- **Key expressions or variables used:**  
  - `={{ $json.Keyword + ' in ' + $json.Location }}`
- **Input and output connections:**  
  - Input: `Read Search Requests`
  - Output: `Format Business Results`
- **Version-specific requirements:**  
  Uses `typeVersion: 4.3`.
- **Edge cases or potential failure types:**  
  - Missing RapidAPI authentication headers
  - API quota exhaustion or billing limits
  - Rate limiting when multiple sheet rows are processed
  - Empty or malformed `Keyword` / `Location` values
  - API returning a structure different from the expected `data` array
  - Timeouts for slow API responses
- **Sub-workflow reference:**  
  None.

---

## 2.3 Result Normalization

### Overview
This block restructures the API payload into one lead item per business with simplified fields expected by the AI scoring and storage layers. It also injects the originating search ID and fallback placeholder values for missing data.

### Nodes Involved
- `Format Business Results`

### Node Details

#### Format Business Results
- **Type and technical role:** `n8n-nodes-base.code`  
  Runs JavaScript to transform the API response into normalized lead objects.
- **Configuration choices:**  
  The code:
  - Reads `data` from the first input item
  - Reads `ID` from the first input item as `searchId`
  - Loops through each returned business
  - Emits one item per business with:
    - `Search ID`
    - `Business Name`
    - `Phone`
    - `Email`
    - `Address`
    - `Website`
- **Key expressions or variables used:**  
  The JavaScript relies on:
  - `$input.first().json.data || []`
  - `$input.first().json.ID`
  - `b.name`
  - `b.phone_number`
  - `b.emails_and_contacts.emails[0]`
  - `b.full_address`
  - `b.website`
- **Important implementation note:**  
  The code uses placeholder values rather than blanks for missing fields:
  - Business Name → `'d'`
  - Phone → `'a'`
  - Email → `'m'`
  - Address → `'n'`
  - Website → `'n'`
  
  These placeholders may affect AI interpretation and lead quality scoring.
- **Input and output connections:**  
  - Input: `Search Businesses API`
  - Output: `Message a model`
- **Version-specific requirements:**  
  Uses `typeVersion: 2`.
- **Edge cases or potential failure types:**  
  - If the API response shape changes, `data` may be undefined
  - `$input.first().json.ID` may not exist if the previous node does not preserve source row context as expected
  - Placeholder values can be mistaken as real values by downstream logic
  - If a business has multiple emails, only the first is kept
- **Sub-workflow reference:**  
  None.

---

## 2.4 AI-Based Lead Qualification

### Overview
This block uses OpenAI to analyze each formatted business and convert basic contact details into sales-relevant qualification fields. It attempts to produce structured JSON suitable for direct insertion into the results sheet.

### Nodes Involved
- `Message a model`

### Node Details

#### Message a model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.openAi`  
  Sends a chat-style prompt to an OpenAI model for classification and lead scoring.
- **Configuration choices:**  
  - Model: `gpt-4.1-mini`
  - Uses a system message defining the AI as a B2B lead generation analyst
  - Uses a user message containing business details and a strict JSON schema
  - Requires output fields:
    - `category`
    - `description`
    - `lead_score`
    - `confidence`
    - `outreach_line`
- **Key expressions or variables used:**  
  In the user prompt:
  - `{{$json["Business Name"]}}`
  - `{{$json["Website"]}}`
  - `{{$json["Address"]}}`
  - `{{$json["Phone"]}}`
  - `{{$json["Email"]}}`
- **Input and output connections:**  
  - Input: `Format Business Results`
  - Output: `Write to Business Results`
- **Version-specific requirements:**  
  Uses `typeVersion: 2.1`. This requires the LangChain/OpenAI integration node available in the installed n8n version and valid OpenAI credentials.
- **Edge cases or potential failure types:**  
  - OpenAI credential/authentication errors
  - Model unavailability or account access restrictions for `gpt-4.1-mini`
  - Output not matching the requested JSON schema exactly
  - Token or rate limits if many businesses are processed
  - Placeholder values like `a`, `m`, `n`, `d` degrading classification quality
- **Sub-workflow reference:**  
  None.

### Important downstream mapping warning
The next node expects top-level fields named:

- `Category`
- `Description`
- `LeadScore`
- `Confidence`
- `OutreachLine`

However, the AI prompt requests lowercase JSON keys:

- `category`
- `description`
- `lead_score`
- `confidence`
- `outreach_line`

Unless the OpenAI node is configured to parse and remap these outputs automatically, the Google Sheets write step may receive empty fields for the AI-enriched columns. This is the most important implementation risk in the workflow.

---

## 2.5 Result Persistence

### Overview
This block writes enriched lead records into a Google Sheets output sheet. It maps both contact fields and AI analysis fields into spreadsheet columns and uses append-or-update behavior.

### Nodes Involved
- `Write to Business Results`

### Node Details

#### Write to Business Results
- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Persists final lead records into a target spreadsheet.
- **Configuration choices:**  
  - Operation: `appendOrUpdate`
  - Target worksheet identified by numeric sheet value `72104887`
  - Mapping mode: define fields manually
  - Matching columns: `Search ID`
  - Mapped columns include:
    - `Search ID`
    - `Business Name`
    - `Phone`
    - `Email`
    - `Address`
    - `Website`
    - `Category`
    - `Description`
    - `LeadScore`
    - `Confidence`
    - `OutreachLine`
- **Key expressions or variables used:**  
  - `={{ $json.Email }}`
  - `={{ $json.Phone }}`
  - `={{ $json.Address }}`
  - `={{ $json.Website }}`
  - `={{$json.Category}}`
  - `={{$json.LeadScore}}`
  - `={{ $('Read Search Requests').item.json.ID }}`
  - `={{$json.Confidence}}`
  - `={{$json.Description}}`
  - `={{$json.OutreachLine}}`
  - `={{ $json['Business Name'] }}`
- **Important implementation note:**  
  The node matches only on `Search ID`. Because many businesses can share the same originating search ID, `appendOrUpdate` on that single field may overwrite prior businesses from the same search instead of preserving one row per business. A more stable matching key would likely be a combination such as:
  - `Search ID` + `Business Name`
  - or a generated unique business identifier
- **Input and output connections:**  
  - Input: `Message a model`
  - Output: none
- **Version-specific requirements:**  
  Uses `typeVersion: 4.7`.
- **Edge cases or potential failure types:**  
  - Google authentication failure
  - Output sheet schema mismatch
  - AI fields arriving under unexpected key names
  - Expression resolution issues with `$('Read Search Requests').item.json.ID` if item pairing is lost
  - Duplicate/update collisions because `Search ID` is not unique per business
- **Sub-workflow reference:**  
  None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Trigger | n8n-nodes-base.scheduleTrigger | Runs the workflow automatically on a recurring schedule |  | Read Search Requests | ### Automated local business lead generation and AI-powered lead scoring<br>This workflow helps you automatically discover, enrich, and qualify local business leads using Google Sheets, RapidAPI, and OpenAI.<br>**What it does:**<br>* Reads keywords and locations from Google Sheets.<br>* Finds local businesses with contact details via API.<br>* Extracts email, phone, website, and address.<br>* Uses AI to classify businesses and assign lead scores.<br>* Generates short outreach messages.<br>* Stores enriched leads back in Google Sheets.<br>**How it works:**<br>1. Schedule Trigger runs automatically.<br>2. Reads search inputs from Google Sheets.<br>3. Calls API to fetch business data.<br>4. Formats results into structured fields.<br>5. OpenAI analyzes and scores each lead.<br>6. Saves results to a Google Sheets database.<br>**Setup notes:**<br>* Add your RapidAPI key in the HTTP Request node.<br>* Connect Google Sheets and OpenAI credentials.<br>* Ensure input sheet includes Keyword, Location, and ID.<br>* Avoid hardcoding credentials.<br>* Use a Set node for easy customization.<br>**Step 1 – Schedule Trigger**<br>Runs the workflow automatically at set intervals. |
| Read Search Requests | n8n-nodes-base.googleSheets | Reads keyword/location search requests from Google Sheets | Schedule Trigger | Search Businesses API | ### Automated local business lead generation and AI-powered lead scoring<br>This workflow helps you automatically discover, enrich, and qualify local business leads using Google Sheets, RapidAPI, and OpenAI.<br>**What it does:**<br>* Reads keywords and locations from Google Sheets.<br>* Finds local businesses with contact details via API.<br>* Extracts email, phone, website, and address.<br>* Uses AI to classify businesses and assign lead scores.<br>* Generates short outreach messages.<br>* Stores enriched leads back in Google Sheets.<br>**How it works:**<br>1. Schedule Trigger runs automatically.<br>2. Reads search inputs from Google Sheets.<br>3. Calls API to fetch business data.<br>4. Formats results into structured fields.<br>5. OpenAI analyzes and scores each lead.<br>6. Saves results to a Google Sheets database.<br>**Setup notes:**<br>* Add your RapidAPI key in the HTTP Request node.<br>* Connect Google Sheets and OpenAI credentials.<br>* Ensure input sheet includes Keyword, Location, and ID.<br>* Avoid hardcoding credentials.<br>* Use a Set node for easy customization.<br>**Step 2 – Read Search Requests**<br>Fetches keywords and locations from Google Sheets. |
| Search Businesses API | n8n-nodes-base.httpRequest | Queries the RapidAPI local business search endpoint | Read Search Requests | Format Business Results | ### Automated local business lead generation and AI-powered lead scoring<br>This workflow helps you automatically discover, enrich, and qualify local business leads using Google Sheets, RapidAPI, and OpenAI.<br>**What it does:**<br>* Reads keywords and locations from Google Sheets.<br>* Finds local businesses with contact details via API.<br>* Extracts email, phone, website, and address.<br>* Uses AI to classify businesses and assign lead scores.<br>* Generates short outreach messages.<br>* Stores enriched leads back in Google Sheets.<br>**How it works:**<br>1. Schedule Trigger runs automatically.<br>2. Reads search inputs from Google Sheets.<br>3. Calls API to fetch business data.<br>4. Formats results into structured fields.<br>5. OpenAI analyzes and scores each lead.<br>6. Saves results to a Google Sheets database.<br>**Setup notes:**<br>* Add your RapidAPI key in the HTTP Request node.<br>* Connect Google Sheets and OpenAI credentials.<br>* Ensure input sheet includes Keyword, Location, and ID.<br>* Avoid hardcoding credentials.<br>* Use a Set node for easy customization.<br>**Step 3 – Search Businesses API**<br>Retrieves local business data with contact details. |
| Format Business Results | n8n-nodes-base.code | Normalizes API response into one item per business | Search Businesses API | Message a model | ### Automated local business lead generation and AI-powered lead scoring<br>This workflow helps you automatically discover, enrich, and qualify local business leads using Google Sheets, RapidAPI, and OpenAI.<br>**What it does:**<br>* Reads keywords and locations from Google Sheets.<br>* Finds local businesses with contact details via API.<br>* Extracts email, phone, website, and address.<br>* Uses AI to classify businesses and assign lead scores.<br>* Generates short outreach messages.<br>* Stores enriched leads back in Google Sheets.<br>**How it works:**<br>1. Schedule Trigger runs automatically.<br>2. Reads search inputs from Google Sheets.<br>3. Calls API to fetch business data.<br>4. Formats results into structured fields.<br>5. OpenAI analyzes and scores each lead.<br>6. Saves results to a Google Sheets database.<br>**Setup notes:**<br>* Add your RapidAPI key in the HTTP Request node.<br>* Connect Google Sheets and OpenAI credentials.<br>* Ensure input sheet includes Keyword, Location, and ID.<br>* Avoid hardcoding credentials.<br>* Use a Set node for easy customization.<br>**Step 4 – Format Business Results**<br>Cleans and structures API response into usable fields. |
| Message a model | @n8n/n8n-nodes-langchain.openAi | Uses OpenAI to classify and score each business lead | Format Business Results | Write to Business Results | ### Automated local business lead generation and AI-powered lead scoring<br>This workflow helps you automatically discover, enrich, and qualify local business leads using Google Sheets, RapidAPI, and OpenAI.<br>**What it does:**<br>* Reads keywords and locations from Google Sheets.<br>* Finds local businesses with contact details via API.<br>* Extracts email, phone, website, and address.<br>* Uses AI to classify businesses and assign lead scores.<br>* Generates short outreach messages.<br>* Stores enriched leads back in Google Sheets.<br>**How it works:**<br>1. Schedule Trigger runs automatically.<br>2. Reads search inputs from Google Sheets.<br>3. Calls API to fetch business data.<br>4. Formats results into structured fields.<br>5. OpenAI analyzes and scores each lead.<br>6. Saves results to a Google Sheets database.<br>**Setup notes:**<br>* Add your RapidAPI key in the HTTP Request node.<br>* Connect Google Sheets and OpenAI credentials.<br>* Ensure input sheet includes Keyword, Location, and ID.<br>* Avoid hardcoding credentials.<br>* Use a Set node for easy customization.<br>**Step 5 – Message a model (OpenAI)**<br>Classifies leads, assigns scores, and generates outreach lines. |
| Write to Business Results | n8n-nodes-base.googleSheets | Writes enriched lead records into the output Google Sheet | Message a model |  | ### Automated local business lead generation and AI-powered lead scoring<br>This workflow helps you automatically discover, enrich, and qualify local business leads using Google Sheets, RapidAPI, and OpenAI.<br>**What it does:**<br>* Reads keywords and locations from Google Sheets.<br>* Finds local businesses with contact details via API.<br>* Extracts email, phone, website, and address.<br>* Uses AI to classify businesses and assign lead scores.<br>* Generates short outreach messages.<br>* Stores enriched leads back in Google Sheets.<br>**How it works:**<br>1. Schedule Trigger runs automatically.<br>2. Reads search inputs from Google Sheets.<br>3. Calls API to fetch business data.<br>4. Formats results into structured fields.<br>5. OpenAI analyzes and scores each lead.<br>6. Saves results to a Google Sheets database.<br>**Setup notes:**<br>* Add your RapidAPI key in the HTTP Request node.<br>* Connect Google Sheets and OpenAI credentials.<br>* Ensure input sheet includes Keyword, Location, and ID.<br>* Avoid hardcoding credentials.<br>* Use a Set node for easy customization.<br>**Step 6 – Write to Business Results**<br>Stores enriched leads in Google Sheets for outreach. |
| Sticky Note6 | n8n-nodes-base.stickyNote | Visual documentation for the final storage step |  |  |  |
| Sticky Note7 | n8n-nodes-base.stickyNote | Visual overview and setup guidance for the full workflow |  |  |  |
| Sticky Note8 | n8n-nodes-base.stickyNote | Visual label for the schedule step |  |  |  |
| Sticky Note9 | n8n-nodes-base.stickyNote | Visual label for the Google Sheets input step |  |  |  |
| Sticky Note10 | n8n-nodes-base.stickyNote | Visual label for the API search step |  |  |  |
| Sticky Note11 | n8n-nodes-base.stickyNote | Visual label for the formatting step |  |  |  |
| Sticky Note12 | n8n-nodes-base.stickyNote | Visual label for the OpenAI scoring step |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.
   - Name it: `Generate and qualify local business leads using Google Sheets, RapidAPI and OpenAI`.

2. **Add a Schedule Trigger node.**
   - Node type: `Schedule Trigger`
   - Configure an interval-based schedule.
   - Set it to run on a minute-based frequency as desired.
   - This will be the workflow entry point.

3. **Prepare the input Google Sheet.**
   - Create a spreadsheet with at least one worksheet for search requests.
   - Add columns:
     - `ID`
     - `Keyword`
     - `Location`
   - Example rows:
     - `1 | plumbers | Austin, TX`
     - `2 | dentists | Miami, FL`

4. **Add a Google Sheets node named `Read Search Requests`.**
   - Node type: `Google Sheets`
   - Connect it after `Schedule Trigger`.
   - Select Google Sheets credentials.
   - Choose the spreadsheet containing your input data.
   - Select the worksheet corresponding to the request list, such as the first sheet (`gid=0`).
   - Configure it to read rows from the sheet.

5. **Add an HTTP Request node named `Search Businesses API`.**
   - Node type: `HTTP Request`
   - Connect it after `Read Search Requests`.
   - Set URL to:
     - `https://local-business-data.p.rapidapi.com/search`
   - Enable query parameters.
   - Add query parameters:
     - `query` → `={{ $json.Keyword + ' in ' + $json.Location }}`
     - `limit` → `25`
     - `extract_emails_and_contacts` → `true`
   - Enable headers.
   - Add the RapidAPI authentication headers required by your subscribed API, typically:
     - `X-RapidAPI-Key` → your RapidAPI key
     - `X-RapidAPI-Host` → host required by the provider
   - If the API expects GET, keep the default method as GET.

6. **Ensure the HTTP response returns JSON.**
   - The downstream code expects a payload containing a top-level `data` array.
   - Test the node with one sample input row.
   - Confirm returned items include business records with fields such as:
     - `name`
     - `phone_number`
     - `full_address`
     - `website`
     - `emails_and_contacts.emails`

7. **Add a Code node named `Format Business Results`.**
   - Node type: `Code`
   - Connect it after `Search Businesses API`.
   - Set it to JavaScript mode.
   - Paste logic equivalent to the following behavior:
     - Read `data` from the incoming API payload
     - Read `ID` from the originating search request
     - Loop over all businesses
     - Return one n8n item per business with:
       - `Search ID`
       - `Business Name`
       - `Phone`
       - `Email`
       - `Address`
       - `Website`

8. **Use the same field extraction pattern as the original workflow.**
   - Business Name from `b.name`
   - Phone from `b.phone_number`
   - Email from the first email in `b.emails_and_contacts.emails`
   - Address from `b.full_address`
   - Website from `b.website`

9. **Decide whether to keep or improve the fallback values.**
   - Original workflow uses:
     - `'d'` for missing business name
     - `'a'` for missing phone
     - `'m'` for missing email
     - `'n'` for missing address
     - `'n'` for missing website
   - Recommended improvement: use empty strings or explicit labels like `Unknown` instead, to avoid confusing the AI model.

10. **Add an OpenAI node named `Message a model`.**
    - Node type: `OpenAI` via the LangChain/OpenAI integration
    - Connect it after `Format Business Results`.
    - Configure OpenAI credentials.
    - Select model:
      - `gpt-4.1-mini`

11. **Configure the system prompt in `Message a model`.**
    - Add a system message instructing the model to act as a B2B lead generation analyst.
    - Include constraints:
      - be realistic
      - do not hallucinate
      - keep descriptions concise
      - return only valid JSON
      - reduce confidence if data is missing
      - prioritize leads with website and direct contact info

12. **Configure the user prompt in `Message a model`.**
    - Insert expressions for the formatted business fields:
      - `{{$json["Business Name"]}}`
      - `{{$json["Website"]}}`
      - `{{$json["Address"]}}`
      - `{{$json["Phone"]}}`
      - `{{$json["Email"]}}`
    - Ask the model to return this exact JSON structure:
      - `category`
      - `description`
      - `lead_score`
      - `confidence`
      - `outreach_line`

13. **Important: parse and remap the AI output.**
    - The original workflow writes `Category`, `Description`, `LeadScore`, `Confidence`, and `OutreachLine` to Google Sheets.
    - If your OpenAI node returns lowercase keys, add an intermediate `Set` or `Code` node to map:
      - `category` → `Category`
      - `description` → `Description`
      - `lead_score` → `LeadScore`
      - `confidence` → `Confidence`
      - `outreach_line` → `OutreachLine`
    - This mapping is strongly recommended even though it is not present in the JSON.

14. **Prepare the output Google Sheet.**
    - Create or select a worksheet for final lead results.
    - Add columns:
      - `Search ID`
      - `Business Name`
      - `Phone`
      - `Email`
      - `Address`
      - `Website`
      - `Category`
      - `Description`
      - `LeadScore`
      - `Confidence`
      - `OutreachLine`

15. **Add a Google Sheets node named `Write to Business Results`.**
    - Node type: `Google Sheets`
    - Connect it after `Message a model` or after your remapping node if you add one.
    - Choose the output spreadsheet and results worksheet.
    - Set operation to `Append or Update`.

16. **Configure the output field mapping manually.**
    - Map spreadsheet columns to incoming fields:
      - `Business Name` ← `{{ $json['Business Name'] }}`
      - `Phone` ← `{{ $json.Phone }}`
      - `Email` ← `{{ $json.Email }}`
      - `Address` ← `{{ $json.Address }}`
      - `Website` ← `{{ $json.Website }}`
      - `Category` ← `{{ $json.Category }}`
      - `Description` ← `{{ $json.Description }}`
      - `LeadScore` ← `{{ $json.LeadScore }}`
      - `Confidence` ← `{{ $json.Confidence }}`
      - `OutreachLine` ← `{{ $json.OutreachLine }}`
      - `Search ID` ← either:
        - `{{ $json['Search ID'] }}` if preserved directly
        - or `{{ $('Read Search Requests').item.json.ID }}` as in the original workflow

17. **Choose a safe matching strategy for updates.**
    - The original workflow matches only on `Search ID`.
    - Recommended improvement:
      - match on `Search ID` and `Business Name`
      - or introduce a unique ID per business row
    - If you keep only `Search ID`, multiple businesses for the same search may overwrite each other.

18. **Connect the nodes in this order.**
    - `Schedule Trigger` → `Read Search Requests`
    - `Read Search Requests` → `Search Businesses API`
    - `Search Businesses API` → `Format Business Results`
    - `Format Business Results` → `Message a model`
    - `Message a model` → `Write to Business Results`

19. **Add credentials.**
    - **Google Sheets credentials:** OAuth2 or service account, depending on your setup
    - **OpenAI credentials:** API key with access to `gpt-4.1-mini`
    - **RapidAPI credentials:** usually provided as headers in the HTTP Request node

20. **Test the workflow step by step.**
    - Run `Read Search Requests` and verify input rows
    - Run `Search Businesses API` and inspect the JSON shape
    - Run `Format Business Results` and verify one item per business
    - Run `Message a model` and confirm valid structured output
    - Run `Write to Business Results` and verify rows appear correctly in the sheet

21. **Handle common production issues before activation.**
    - Add error handling for API failures
    - Consider rate limiting if many search rows are processed
    - Add deduplication logic if repeated searches are expected
    - Replace placeholder values with empty strings or nulls
    - Add a parsing/remapping node after OpenAI if needed

22. **Activate the workflow** once all tests pass.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Automated local business lead generation and AI-powered lead scoring | Workflow overview note |
| Reads keywords and locations from Google Sheets; finds local businesses with contact details via API; extracts email, phone, website, and address; uses AI to classify businesses and assign lead scores; generates short outreach messages; stores enriched leads back in Google Sheets | Functional summary |
| Setup notes: add your RapidAPI key in the HTTP Request node; connect Google Sheets and OpenAI credentials; ensure input sheet includes Keyword, Location, and ID; avoid hardcoding credentials; use a Set node for easy customization | Implementation guidance |

## Additional implementation notes
- There are **no sub-workflows** in this workflow.
- There is **one entry point**: `Schedule Trigger`.
- The two biggest design risks are:
  1. **AI output key mismatch** between lowercase prompt output and uppercase sheet mapping
  2. **Non-unique update matching** using only `Search ID`
- For a production-grade version, consider inserting:
  - a `Set` or `Code` node after OpenAI for field normalization
  - a deduplication step before writing to Sheets
  - retry/error handling for API and model calls