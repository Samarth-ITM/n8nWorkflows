Auto-fill Monday.com lead boards with FullEnrich People Search results

https://n8nworkflows.xyz/workflows/auto-fill-monday-com-lead-boards-with-fullenrich-people-search-results-14226


# Auto-fill Monday.com lead boards with FullEnrich People Search results

# 1. Workflow Overview

This workflow listens for a Monday.com webhook triggered when a new item is created on a “search criteria” board, uses the submitted criteria to query FullEnrich People Search, then creates one new item per returned person in a separate Monday.com results board.

Typical use case:
- A user creates a request item in Monday.com with fields such as job title, industry, location, company size, and desired number of results.
- Monday.com sends the item payload to this workflow.
- The workflow acknowledges the webhook challenge format expected by Monday.com.
- It performs a FullEnrich people search.
- It splits the returned people array into individual records.
- It creates result items in another Monday.com board.

## 1.1 Input Reception and Monday.com Handshake

The workflow starts with a webhook node that receives a POST request from Monday.com. It immediately returns the required challenge payload through a webhook response node, which is necessary for Monday.com webhook verification.

## 1.2 Lead Search via FullEnrich

After the challenge response step, the workflow sends a POST request to FullEnrich’s `/api/v2/people/search` endpoint. It builds the search request dynamically from values stored in Monday.com item columns.

## 1.3 Result Normalization

The FullEnrich API returns an array of people. A Code node converts that array into individual n8n items so downstream nodes can process each person separately.

## 1.4 Result Creation in Monday.com

For each person returned by FullEnrich, the workflow creates a new board item in Monday.com and populates mapped columns such as first name, last name, title, company, domain, LinkedIn URL, and location.

---

# 2. Block-by-Block Analysis

## Block 1 — Input Reception and Monday.com Handshake

### Overview

This block receives the webhook call from Monday.com and returns the challenge response required during webhook verification. It is the workflow’s entry point and is mandatory for Monday.com to accept the webhook endpoint.

### Nodes Involved

- Monday.com Webhook
- Challenge Response

### Node Details

#### 1. Monday.com Webhook

- **Type and technical role:** `n8n-nodes-base.webhook`  
  Receives incoming HTTP POST requests from Monday.com.
- **Configuration choices:**
  - HTTP method: `POST`
  - Path: `00ffac35-076e-42cd-a794-16cee3805405`
  - Response mode: `responseNode`
- **Key expressions or variables used:**
  - No complex expressions in this node.
  - The incoming payload is later accessed through `$json.body...`
- **Input and output connections:**
  - Input: none, this is the entry point
  - Output: sends data to `Challenge Response`
- **Version-specific requirements:**
  - Uses node type version `2.1`
  - `responseNode` mode requires a dedicated Respond to Webhook node downstream
- **Edge cases or potential failure types:**
  - Wrong webhook URL configured in Monday.com
  - Monday.com sending a payload format different from expected
  - If the workflow is inactive, production webhook calls will fail
  - If using the test URL instead of the production URL in Monday.com automation, production runs will not work as expected
- **Sub-workflow reference:** none

#### 2. Challenge Response

- **Type and technical role:** `n8n-nodes-base.respondToWebhook`  
  Sends a JSON response back to Monday.com containing the challenge value.
- **Configuration choices:**
  - Respond with: `json`
  - Response body:
    - `={{ JSON.stringify({ challenge: $json.body.challenge }) }}`
  - Response header:
    - `Content-Type: application/json`
- **Key expressions or variables used:**
  - `$json.body.challenge`
- **Input and output connections:**
  - Input: `Monday.com Webhook`
  - Output: `FullEnrich People Search`
- **Version-specific requirements:**
  - Uses node type version `1.5`
  - Works in conjunction with the Webhook node configured in `responseNode` mode
- **Edge cases or potential failure types:**
  - If `body.challenge` is absent, the response will contain `undefined`
  - Monday.com may reject verification if the response format does not exactly match expectations
  - In some integrations, challenge handling should be conditional; here it is always returned, regardless of whether the request is a verification request or an event payload
- **Sub-workflow reference:** none

**Important implementation note:**  
This workflow responds with a challenge payload for every incoming request, then continues processing. That may work in some contexts, but some Monday.com webhook patterns distinguish between:
- an initial verification request containing `challenge`
- actual event requests containing event data

If Monday.com event requests do not contain `challenge`, this node may reply with `{ "challenge": undefined }`. The workflow may still continue internally, but you should validate this behavior in your environment.

---

## Block 2 — Lead Search via FullEnrich

### Overview

This block calls FullEnrich People Search using search criteria extracted from the triggering Monday.com item. It converts user-entered board values into the JSON request expected by FullEnrich.

### Nodes Involved

- FullEnrich People Search

### Node Details

#### 3. FullEnrich People Search

- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Sends an authenticated POST request to FullEnrich’s People Search API.
- **Configuration choices:**
  - Method: `POST`
  - URL: `https://app.fullenrich.com/api/v2/people/search`
  - Authentication: predefined credential type
  - Credential type: `fullEnrichApi`
  - Sends body: yes
  - Content type: raw JSON
  - Header:
    - `Content-Type: application/json`
- **Key expressions or variables used:**
  - Result count:
    - `{{ $json.body.event.columnValues.YOUR_NUMBER_COLUMN_ID.value }}`
  - Job title:
    - `{{ $json.body.event.columnValues.YOUR_JOB_TITLE_COLUMN_ID.value }}`
  - Industry:
    - `{{ $json.body.event.columnValues.YOUR_INDUSTRY_COLUMN_ID.value }}`
  - Location:
    - `{{ $json.body.event.columnValues.YOUR_LOCATION_COLUMN_ID.value }}`
  - Company size:
    - `{{ $json.body.event.columnValues.YOUR_COMPANY_SIZE_COLUMN_ID.value }}`
- **Interpreted request structure:**
  - `offset: 0`
  - `limit`: number of results requested from Monday.com
  - `job_titles`: one entry, fuzzy match (`exact_match: false`)
  - `industries`: one entry, fuzzy match
  - `locations`: one entry, fuzzy match
  - `current_company_headcounts`: one entry, exact match (`exact_match: true`)
- **Input and output connections:**
  - Input: `Challenge Response`
  - Output: `Split Results`
- **Version-specific requirements:**
  - Uses node type version `4.4`
  - Requires a configured `fullEnrichApi` credential in n8n
- **Edge cases or potential failure types:**
  - Missing or incorrect FullEnrich credentials
  - Invalid column IDs left as placeholders
  - Monday.com payload values may be empty, malformed, or differently typed than expected
  - `limit` may be invalid if the Monday.com column does not hold a numeric value
  - FullEnrich API may reject unsupported company size formats
  - Network failure, API downtime, rate limiting, or non-2xx responses
  - If `body.event.columnValues` structure differs from expected Monday.com payload shape, expressions will fail or return empty strings
- **Sub-workflow reference:** none

**Important configuration note:**  
All placeholders such as `YOUR_NUMBER_COLUMN_ID` must be replaced with real Monday.com column IDs from the search criteria board. Without that, the request body will not be populated correctly.

---

## Block 3 — Result Normalization

### Overview

This block transforms the `people` array returned by FullEnrich into one n8n item per person. This is required because the Monday.com node expects to process records individually.

### Nodes Involved

- Split Results

### Node Details

#### 4. Split Results

- **Type and technical role:** `n8n-nodes-base.code`  
  Executes JavaScript to split an array into separate workflow items.
- **Configuration choices:**
  - JavaScript:
    - `return $input.first().json.people.map(person => ({ json: person }));`
- **Key expressions or variables used:**
  - `$input.first().json.people`
- **Input and output connections:**
  - Input: `FullEnrich People Search`
  - Output: `Create Monday.com Items`
- **Version-specific requirements:**
  - Uses node type version `2`
- **Edge cases or potential failure types:**
  - If the API response has no `people` property, the code throws an error
  - If `people` is not an array, `.map()` throws an error
  - If the API returns an empty array, the node returns no items and nothing is created in Monday.com
- **Sub-workflow reference:** none

**Safer alternative recommendation:**  
A more defensive version would validate the response before mapping, for example by checking `Array.isArray(...)`.

---

## Block 4 — Result Creation in Monday.com

### Overview

This block creates one Monday.com board item per FullEnrich person. It maps person and employment fields to the destination board’s columns.

### Nodes Involved

- Create Monday.com Items

### Node Details

#### 5. Create Monday.com Items

- **Type and technical role:** `n8n-nodes-base.mondayCom`  
  Creates items in a Monday.com board.
- **Configuration choices:**
  - Resource: `boardItem`
  - Item name:
    - `={{ $json.full_name }}`
  - Additional field `columnValues` set as a JSON object string
- **Key expressions or variables used:**
  - Item name:
    - `$json.full_name`
  - First name:
    - `$json.first_name`
  - Last name:
    - `$json.last_name`
  - Current title:
    - `$json.employment.current.title`
  - Company name:
    - `$json.employment.current.company.name`
  - Company domain:
    - `$json.employment.current.company.domain`
  - LinkedIn URL:
    - `$json.social_profiles.linkedin.url`
  - Location:
    - `{{ $json.location.city ?? '' }}, {{ $json.location.country ?? '' }}`
- **Interpreted mapping behavior:**
  - Creates a Monday.com item named after the person’s full name
  - Writes selected FullEnrich fields into target board columns
- **Input and output connections:**
  - Input: `Split Results`
  - Output: none
- **Version-specific requirements:**
  - Uses node type version `1`
  - Requires Monday.com credentials in n8n
  - Also requires destination board ID and optionally group ID to be configured in the node, although those are not visible in the shared summary and are referenced by sticky notes
- **Edge cases or potential failure types:**
  - Missing Monday.com credentials
  - Destination board ID/group ID not configured
  - Placeholder column IDs not replaced
  - Some people may not have `employment.current`, `company`, `linkedin`, or `location` fields
  - Nested expressions like `$json.social_profiles.linkedin.url` may fail if an intermediate object is missing
  - Monday.com column types may reject values if format does not match the column definition
  - If duplicate items are undesirable, this workflow has no deduplication logic
- **Sub-workflow reference:** none

**Important implementation note:**  
This node assumes a fairly complete person object from FullEnrich. If records are incomplete, use null-safe expressions or a pre-processing node before item creation.

---

## Block 5 — Documentation and Setup Notes

### Overview

This workflow includes several sticky notes that document the flow, setup requirements, and purpose of each step. They do not execute logic but are important for maintenance and reproduction.

### Nodes Involved

- Sticky Note
- Sticky Note1
- Sticky Note2
- Sticky Note3
- Sticky Note4
- Sticky Note5
- Sticky Note6

### Node Details

#### 6. Sticky Note

- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Canvas documentation note summarizing the workflow.
- **Configuration choices:**
  - Describes end-to-end purpose and flow
- **Input and output connections:** none
- **Version-specific requirements:** version `1`
- **Edge cases or potential failure types:** none
- **Sub-workflow reference:** none

#### 7. Sticky Note1

- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:**
  - Setup instructions covering boards, automation, column IDs, board/group IDs, credentials, and activation
- **Input and output connections:** none
- **Version-specific requirements:** version `1`
- **Edge cases or potential failure types:** none
- **Sub-workflow reference:** none

#### 8. Sticky Note2

- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:**
  - Labels the webhook step
- **Input and output connections:** none
- **Version-specific requirements:** version `1`
- **Edge cases or potential failure types:** none
- **Sub-workflow reference:** none

#### 9. Sticky Note3

- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:**
  - Labels the challenge response step
- **Input and output connections:** none
- **Version-specific requirements:** version `1`
- **Edge cases or potential failure types:** none
- **Sub-workflow reference:** none

#### 10. Sticky Note4

- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:**
  - Labels the FullEnrich People Search step
- **Input and output connections:** none
- **Version-specific requirements:** version `1`
- **Edge cases or potential failure types:** none
- **Sub-workflow reference:** none

#### 11. Sticky Note5

- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:**
  - Labels the result splitting step
- **Input and output connections:** none
- **Version-specific requirements:** version `1`
- **Edge cases or potential failure types:** none
- **Sub-workflow reference:** none

#### 12. Sticky Note6

- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:**
  - Labels the Monday.com item creation step
- **Input and output connections:** none
- **Version-specific requirements:** version `1`
- **Edge cases or potential failure types:** none
- **Sub-workflow reference:** none

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Monday.com Webhook | Webhook | Receives POST events from Monday.com |  | Challenge Response | ## 1️⃣ Webhook<br>Receives a POST from Monday.com when a new item is created on your search criteria board. |
| Challenge Response | Respond to Webhook | Returns the Monday.com challenge JSON response | Monday.com Webhook | FullEnrich People Search | ## 2️⃣ Challenge<br>Responds to Monday.com's webhook challenge handshake (required for verification). |
| FullEnrich People Search | HTTP Request | Searches people in FullEnrich using Monday.com criteria | Challenge Response | Split Results | ## 3️⃣ People Search<br>Calls FullEnrich API with job title, industry, location, and company size from your Monday.com columns. |
| Split Results | Code | Splits FullEnrich people array into one item per person | FullEnrich People Search | Create Monday.com Items | ## 4️⃣ Split<br>Splits the array of people results into individual items for Monday.com. |
| Create Monday.com Items | Monday.com | Creates one Monday.com item per FullEnrich person | Split Results |  | ## 5️⃣ Create Items<br>Creates a new Monday.com item for each person found, with name, title, company, domain, LinkedIn, and location. |
| Sticky Note | Sticky Note | Visual documentation of the overall workflow |  |  | ## 🚀 Auto-fill Monday.com with FullEnrich People Search<br><br>This workflow searches for leads using FullEnrich's People Search API based on criteria from a Monday.com board, then creates items with the results on a second board.<br><br>**Flow:** Monday.com webhook → challenge response → FullEnrich People Search → split results → create Monday.com items |
| Sticky Note1 | Sticky Note | Visual setup instructions |  |  | ## ⚙️ Setup<br><br>1. **Search Criteria board** — Create a Monday.com board with columns: Job Title, Industry, Location, Company Size, Number of Results<br>2. **Results board** — Create a board with: First Name, Last Name, Job Title, Company, Domain, LinkedIn, Location<br>3. **Monday.com automation** — On the search board: *When item created → send webhook* to this workflow's **production URL**<br>4. **Column IDs** — Replace all `YOUR_*_COLUMN_ID` placeholders in the HTTP Request and Monday.com nodes with your actual column IDs<br>5. Set the **board ID** and **group ID** in the "Create Monday.com Items" node<br>6. Connect **FullEnrich** and **Monday.com** credentials<br>7. Activate the workflow |
| Sticky Note2 | Sticky Note | Visual label for the webhook step |  |  | ## 1️⃣ Webhook<br>Receives a POST from Monday.com when a new item is created on your search criteria board. |
| Sticky Note3 | Sticky Note | Visual label for the challenge step |  |  | ## 2️⃣ Challenge<br>Responds to Monday.com's webhook challenge handshake (required for verification). |
| Sticky Note4 | Sticky Note | Visual label for the people search step |  |  | ## 3️⃣ People Search<br>Calls FullEnrich API with job title, industry, location, and company size from your Monday.com columns. |
| Sticky Note5 | Sticky Note | Visual label for the split step |  |  | ## 4️⃣ Split<br>Splits the array of people results into individual items for Monday.com. |
| Sticky Note6 | Sticky Note | Visual label for the create-items step |  |  | ## 5️⃣ Create Items<br>Creates a new Monday.com item for each person found, with name, title, company, domain, LinkedIn, and location. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n**
   - Name it something like: `Auto-fill Monday.com with leads from FullEnrich People Search`.

2. **Add a Webhook node**
   - Node type: `Webhook`
   - Name: `Monday.com Webhook`
   - HTTP Method: `POST`
   - Path: choose a unique path, for example the UUID-like path used here
   - Response Mode: `Using Respond to Webhook Node`
   - Save the node

3. **Add a Respond to Webhook node**
   - Node type: `Respond to Webhook`
   - Name: `Challenge Response`
   - Connect `Monday.com Webhook` → `Challenge Response`
   - Set:
     - Respond With: `JSON`
     - Response Headers:
       - `Content-Type` = `application/json`
     - Response Body:
       ```js
       ={{ JSON.stringify({ challenge: $json.body.challenge }) }}
       ```

4. **Add an HTTP Request node**
   - Node type: `HTTP Request`
   - Name: `FullEnrich People Search`
   - Connect `Challenge Response` → `FullEnrich People Search`
   - Configure:
     - Method: `POST`
     - URL: `https://app.fullenrich.com/api/v2/people/search`
     - Authentication: use predefined credential type
     - Credential type: `FullEnrich API`
     - Send Headers: enabled
     - Header:
       - `Content-Type` = `application/json`
     - Send Body: enabled
     - Body Content Type: `Raw`
     - Raw Content Type: `application/json`

5. **Configure FullEnrich credentials**
   - In n8n credentials, create or select a `FullEnrich API` credential
   - Enter the API key or authentication details required by FullEnrich
   - Attach this credential to the `FullEnrich People Search` node

6. **Set the HTTP Request body**
   - Use an expression-based raw JSON body structured like this:
     ```json
     {
       "offset": 0,
       "limit": {{ $json.body.event.columnValues.YOUR_NUMBER_COLUMN_ID.value }},
       "job_titles": [
         {
           "value": "{{ $json.body.event.columnValues.YOUR_JOB_TITLE_COLUMN_ID.value }}",
           "exact_match": false,
           "exclude": false
         }
       ],
       "industries": [
         {
           "value": "{{ $json.body.event.columnValues.YOUR_INDUSTRY_COLUMN_ID.value }}",
           "exact_match": false,
           "exclude": false
         }
       ],
       "locations": [
         {
           "value": "{{ $json.body.event.columnValues.YOUR_LOCATION_COLUMN_ID.value }}",
           "exact_match": false,
           "exclude": false
         }
       ],
       "current_company_headcounts": [
         {
           "value": "{{ $json.body.event.columnValues.YOUR_COMPANY_SIZE_COLUMN_ID.value }}",
           "exact_match": true,
           "exclude": false
         }
       ]
     }
     ```
   - Replace all `YOUR_*_COLUMN_ID` placeholders with real Monday.com column IDs from the search criteria board

7. **Prepare the Monday.com search criteria board**
   - In Monday.com, create a board for search requests
   - Add columns for at least:
     - Job Title
     - Industry
     - Location
     - Company Size
     - Number of Results
   - Record the real column IDs, not just the display names

8. **Configure Monday.com automation**
   - In the search criteria board, add an automation:
     - When item created → send webhook
   - Use the workflow’s **production webhook URL**
   - Do not use the test URL for permanent automation

9. **Add a Code node**
   - Node type: `Code`
   - Name: `Split Results`
   - Connect `FullEnrich People Search` → `Split Results`
   - Set JavaScript code:
     ```javascript
     return $input.first().json.people.map(person => ({ json: person }));
     ```

10. **Optionally harden the split logic**
    - Recommended safer code:
      ```javascript
      const people = $input.first().json.people;
      if (!Array.isArray(people)) {
        return [];
      }
      return people.map(person => ({ json: person }));
      ```
    - This avoids a runtime error when the API returns no people array

11. **Add a Monday.com node**
    - Node type: `Monday.com`
    - Name: `Create Monday.com Items`
    - Connect `Split Results` → `Create Monday.com Items`

12. **Configure Monday.com credentials**
    - Create or select a Monday.com credential in n8n
    - Use the appropriate API token or OAuth method supported by your n8n version
    - Attach the credential to the `Create Monday.com Items` node

13. **Configure the Monday.com item creation action**
    - Resource: `Board Item`
    - Operation: create item
    - Set the destination:
      - Board ID: your results board ID
      - Group ID: target group if required
    - Name field:
      ```js
      ={{ $json.full_name }}
      ```

14. **Prepare the results board in Monday.com**
    - Create a second board to receive lead results
    - Add columns for at least:
      - First Name
      - Last Name
      - Job Title
      - Company
      - Domain
      - LinkedIn
      - Location
   - Capture the actual destination column IDs

15. **Set the `columnValues` mapping in the Monday.com node**
    - Use a JSON string expression like:
      ```json
      {
        "YOUR_FIRST_NAME_COL": "{{ $json.first_name }}",
        "YOUR_LAST_NAME_COL": "{{ $json.last_name }}",
        "YOUR_JOB_TITLE_COL": "{{ $json.employment.current.title }}",
        "YOUR_COMPANY_NAME_COL": "{{ $json.employment.current.company.name }}",
        "YOUR_COMPANY_DOMAIN_COL": "{{ $json.employment.current.company.domain }}",
        "YOUR_LINKEDIN_COL": "{{ $json.social_profiles.linkedin.url }}",
        "YOUR_LOCATION_COL": "{{ $json.location.city ?? '' }}, {{ $json.location.country ?? '' }}"
      }
      ```
   - Replace every placeholder with the actual column ID from the results board

16. **Consider null-safe field handling before production**
    - Some FullEnrich profiles may miss:
      - current employment
      - company
      - LinkedIn URL
      - city or country
    - To avoid failures, you may prefer expressions with defaults, or a Set/Code node before item creation

17. **Optionally add notes for maintainability**
    - Add sticky notes documenting:
      - purpose of the workflow
      - setup steps
      - node-by-node intent
    - This matches the original workflow style

18. **Test the workflow**
    - Trigger the Monday.com automation by creating an item in the search criteria board
    - Verify:
      - the webhook is received
      - FullEnrich returns results
      - the code node emits one item per person
      - Monday.com results board receives the new items

19. **Activate the workflow**
    - Once validated, activate it in n8n
    - Ensure Monday.com points to the production webhook URL

20. **Recommended improvements before large-scale use**
    - Add an IF node to distinguish challenge verification from event processing
    - Add error handling for empty or invalid FullEnrich responses
    - Add rate-limit protection or retry logic
    - Add deduplication to avoid creating repeated lead items

### Sub-workflow setup

This workflow does **not** use any sub-workflows, Execute Workflow nodes, or callable child workflows.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Auto-fill Monday.com with FullEnrich People Search | Workflow title / functional label |
| This workflow searches for leads using FullEnrich's People Search API based on criteria from a Monday.com board, then creates items with the results on a second board. | Overall purpose |
| Flow: Monday.com webhook → challenge response → FullEnrich People Search → split results → create Monday.com items | Overall logic summary |
| Search Criteria board — Create a Monday.com board with columns: Job Title, Industry, Location, Company Size, Number of Results | Setup requirement |
| Results board — Create a board with: First Name, Last Name, Job Title, Company, Domain, LinkedIn, Location | Setup requirement |
| Monday.com automation — On the search board: When item created → send webhook to this workflow's production URL | Monday.com setup |
| Column IDs — Replace all `YOUR_*_COLUMN_ID` placeholders in the HTTP Request and Monday.com nodes with your actual column IDs | Mandatory configuration |
| Set the board ID and group ID in the "Create Monday.com Items" node | Mandatory configuration |
| Connect FullEnrich and Monday.com credentials | Credential requirement |
| Activate the workflow | Deployment requirement |

## Additional implementation notes

- The workflow contains a naming mismatch:
  - **Title provided:** `Auto-fill Monday.com lead boards with FullEnrich People Search results`
  - **Workflow JSON name:** `Auto-fill Monday.com with leads from FullEnrich People Search`
  This does not affect execution, but documentation should treat them as the same workflow.

- The workflow is active in the exported JSON:
  - `"active": true`

- Execution order is set to:
  - `v1`

- No external links, videos, or project credits are embedded in the workflow notes.