Route product signals from Notion to Jira and Slack backlogs

https://n8nworkflows.xyz/workflows/route-product-signals-from-notion-to-jira-and-slack-backlogs-14124


# Route product signals from Notion to Jira and Slack backlogs

# 1. Workflow Overview

This workflow, titled **Signal Router**, monitors a Notion database for product/customer signals whose **Route Status** has been set to **Routing**. Once detected, it extracts the signal details, determines the correct destination, creates a corresponding item in Jira or Notion, updates the original Notion record to mark it as routed, and posts a confirmation reply back into the originating Slack thread.

It is designed for teams that centralize incoming product feedback, bug reports, escalations, or prioritization inputs in Notion and need to dispatch them into downstream delivery or tracking systems in a controlled, auditable way.

## 1.1 Trigger and Intake

The workflow starts by polling a Notion database for pages. It then filters for records whose **Route Status** equals **Routing**.

## 1.2 Signal Normalization

Once a valid record is found, a Code node flattens the Notion page properties into a simpler JSON object. This normalized payload is used by all downstream branches.

## 1.3 Routing Engine

A Switch node examines the **Route Destination** field and routes the signal into one of five destinations:

- Jira Bug
- Jira Feature
- RICE+ Backlog
- Customer Health
- Sprint Backlog

## 1.4 Destination-Specific Record Creation

Each branch creates the target item in either Jira or Notion, then parses the response into a common structure containing:

- the original signal data
- a routed reference
- a routed URL
- optional destination-specific IDs

## 1.5 Closed Loop Update and Slack Confirmation

After routing, all branches converge into a common closing sequence:

1. Update the original Notion signal page to **Route Status = Routed**
2. Store a **Routed Reference**
3. Build a Slack thread confirmation message
4. Post the reply to Slack

---

# 2. Block-by-Block Analysis

## 2.1 Trigger and Intake

**Overview:**  
This block polls Notion for candidate records and ensures that only signals intentionally marked for routing continue through the workflow. It acts as the gatekeeper for all downstream processing.

**Nodes Involved:**
- Signal Route Trigger
- Filter Routing Status

### Node Details

#### Signal Route Trigger
- **Type and technical role:** Notion node, polling trigger/query node for database pages
- **Configuration choices:**
  - Resource: `databasePage`
  - Operation: `getAll`
  - Database ID: `31e06bab-3ebe-811b-b204-c5f41b273303`
  - Limit: `10`
  - `simple: false`, so the full Notion page structure is returned
  - Polling enabled
  - Filter type is set to `singleCondition`, though the actual routing condition is enforced in the next IF node
- **Key expressions or variables used:** None inside this node
- **Input and output connections:**
  - No input; acts as entry point
  - Output to **Filter Routing Status**
- **Version-specific requirements:** Uses Notion node `typeVersion 2.2`
- **Edge cases or potential failure types:**
  - Notion authentication failure
  - Database ID invalid or inaccessible
  - Polling may retrieve already processed pages if upstream filtering is too broad
  - Notion API rate limiting if polling frequency is high
- **Sub-workflow reference:** None

#### Filter Routing Status
- **Type and technical role:** IF node for conditional filtering
- **Configuration choices:**
  - Compares `{{$json.properties?.['Route Status']?.select?.name}}` to the string `Routing`
  - Strict type validation enabled
  - Case-sensitive comparison
- **Key expressions or variables used:**
  - `{{$json.properties?.['Route Status']?.select?.name}}`
- **Input and output connections:**
  - Input from **Signal Route Trigger**
  - True output to **Read Signal Details**
  - False output unused
- **Version-specific requirements:** IF node `typeVersion 2`
- **Edge cases or potential failure types:**
  - If `Route Status` is missing or not a select property, the expression resolves to `undefined`
  - Case mismatch such as `routing` or trailing spaces will fail the condition
  - Pages with malformed properties are silently excluded
- **Sub-workflow reference:** None

---

## 2.2 Signal Normalization

**Overview:**  
This block converts the nested Notion page payload into a flat, reusable JSON object. It standardizes field access for all later routing branches and avoids repetitive Notion property parsing.

**Nodes Involved:**
- Read Signal Details

### Node Details

#### Read Signal Details
- **Type and technical role:** Code node for property extraction and flattening
- **Configuration choices:**
  - Reads `properties` from the Notion page
  - Defines helper `getValue(prop)` to extract values from:
    - title
    - rich_text
    - select
    - url
    - date
  - Returns a single flat object with keys such as:
    - `notionPageId`
    - `signal`
    - `dateCaptured`
    - `sourceChannel`
    - `originalMessage`
    - `author`
    - `signalType`
    - `sentiment`
    - `urgency`
    - `aiAnalysis`
    - `customerMentioned`
    - `routeDestination`
    - `routeStatus`
    - `threadUrl`
    - `slackMessageId`
- **Key expressions or variables used:**
  - `const props = $json.properties;`
  - `getValue(props['Signal'])`, etc.
- **Input and output connections:**
  - Input from **Filter Routing Status**
  - Output to **Route Destination**
- **Version-specific requirements:** Code node `typeVersion 2`
- **Edge cases or potential failure types:**
  - Unsupported Notion property types are returned as empty strings
  - Multi-select, people, relation, checkbox, number, and formula fields are not handled
  - If property names change in Notion, extracted values become blank
  - Missing `properties` object would cause runtime failure
- **Sub-workflow reference:** None

---

## 2.3 Routing Engine

**Overview:**  
This block decides where each signal should be sent based on the normalized `routeDestination` value. It uses a five-way switch with one branch per supported destination.

**Nodes Involved:**
- Route Destination

### Node Details

#### Route Destination
- **Type and technical role:** Switch node for branching logic
- **Configuration choices:**
  - Evaluates `{{$json.routeDestination}}`
  - Five string-equality branches:
    1. `Jira Bug`
    2. `Jira Feature`
    3. `RICE+ Backlog`
    4. `Customer Health`
    5. `Sprint Backlog`
- **Key expressions or variables used:**
  - `{{$json.routeDestination}}`
- **Input and output connections:**
  - Input from **Read Signal Details**
  - Outputs to:
    - **Create Jira Bug**
    - **Create Jira Feature**
    - **Create RICE+ Entry**
    - **Create Customer Health Entry**
    - **Create Sprint Backlog Item**
- **Version-specific requirements:** Switch node `typeVersion 3.2`
- **Edge cases or potential failure types:**
  - Exact match required; any typo or formatting mismatch causes no branch to run
  - No default/fallback branch exists
  - Signals with blank or unexpected route destinations will stall silently
- **Sub-workflow reference:** None

---

## 2.4 Jira Bug Branch

**Overview:**  
This branch creates a Jira Bug issue from the signal and transforms the Jira API response into the common routed payload used by the closing sequence.

**Nodes Involved:**
- Create Jira Bug
- Parse Jira Bug Response

### Node Details

#### Create Jira Bug
- **Type and technical role:** HTTP Request node calling Jira REST API
- **Configuration choices:**
  - Method: `POST`
  - URL: `https://company.atlassian.net/rest/api/3/issue`
  - Authentication: generic credential type using HTTP Basic Auth
  - Sends JSON body
  - Header `Content-Type: application/json`
  - Jira issue payload includes:
    - Project key: `PROJ`
    - Issue type: `Bug`
    - Summary: `[Signal] {{$json.signal}}`
    - Atlassian Document Format description with:
      - heading
      - source channel and author
      - original message
      - AI analysis
      - link to Slack thread
    - Priority mapped from urgency:
      - Critical → Highest
      - High → High
      - Medium → Medium
      - else → Low
    - Labels: `signal-catcher`, `slack-signal`
- **Key expressions or variables used:**
  - `$json.signal`
  - `$json.sourceChannel`
  - `$json.author`
  - `$json.originalMessage`
  - `$json.aiAnalysis`
  - `$json.threadUrl`
  - `$json.urgency`
- **Input and output connections:**
  - Input from **Route Destination**
  - Output to **Parse Jira Bug Response**
- **Version-specific requirements:** HTTP Request node `typeVersion 4.2`
- **Edge cases or potential failure types:**
  - Jira auth failure
  - Invalid project key or issue type name
  - Invalid ADF payload structure
  - Priority name mismatch with Jira instance configuration
  - Very long summaries or descriptions could be rejected
  - Missing required custom fields in Jira could cause 400 errors
- **Sub-workflow reference:** None

#### Parse Jira Bug Response
- **Type and technical role:** Code node to normalize Jira response
- **Configuration choices:**
  - Reads `key` from Jira response
  - Falls back to `PROJ-???` if missing
  - Retrieves original signal payload from `$('Read Signal Details').first().json`
  - Returns merged object plus:
    - `routedReference`
    - `routedUrl`
- **Key expressions or variables used:**
  - `$json.key`
  - `$('Read Signal Details').first().json`
- **Input and output connections:**
  - Input from **Create Jira Bug**
  - Output to **Update Signal Status**
- **Version-specific requirements:** Code node `typeVersion 2`
- **Edge cases or potential failure types:**
  - If the referenced node name changes, the expression breaks
  - If Jira response format changes and `key` is absent, fallback value may hide failures
  - Hardcoded browse URL assumes the same Jira site base URL
- **Sub-workflow reference:** None

---

## 2.5 Jira Feature Branch

**Overview:**  
This branch creates a Jira Story representing a feature-oriented product signal, then converts the Jira response into the common closing format.

**Nodes Involved:**
- Create Jira Feature
- Parse Jira Feature Response

### Node Details

#### Create Jira Feature
- **Type and technical role:** HTTP Request node calling Jira REST API
- **Configuration choices:**
  - Method: `POST`
  - URL: `https://company.atlassian.net/rest/api/3/issue`
  - Authentication: generic credential type with HTTP Basic Auth
  - Sends JSON body with:
    - Project key: `PROJ`
    - Issue type: `Story`
    - Summary: `[Signal] {{$json.signal}}`
    - ADF description including:
      - source channel and author
      - customer mentioned
      - original message
      - AI analysis
      - Slack thread link
    - Priority mapping:
      - Critical → Highest
      - High → High
      - else → Medium
    - Labels: `signal-catcher`, `feature-signal`, `slack-signal`
- **Key expressions or variables used:**
  - `$json.signal`
  - `$json.sourceChannel`
  - `$json.author`
  - `$json.customerMentioned`
  - `$json.originalMessage`
  - `$json.aiAnalysis`
  - `$json.threadUrl`
  - `$json.urgency`
- **Input and output connections:**
  - Input from **Route Destination**
  - Output to **Parse Jira Feature Response**
- **Version-specific requirements:** HTTP Request node `typeVersion 4.2`
- **Edge cases or potential failure types:**
  - Same Jira REST/API/auth risks as the bug branch
  - Priority mapping ignores explicit `Low`; all non-Critical/non-High values become `Medium`
  - Issue type `Story` must exist in the Jira project
- **Sub-workflow reference:** None

#### Parse Jira Feature Response
- **Type and technical role:** Code node to normalize Jira feature creation response
- **Configuration choices:**
  - Extracts Jira key
  - Reattaches original signal data from **Read Signal Details**
  - Adds `routedReference` and `routedUrl`
- **Key expressions or variables used:**
  - `$json.key`
  - `$('Read Signal Details').first().json`
- **Input and output connections:**
  - Input from **Create Jira Feature**
  - Output to **Update Signal Status**
- **Version-specific requirements:** Code node `typeVersion 2`
- **Edge cases or potential failure types:**
  - Same as the bug parse node
- **Sub-workflow reference:** None

---

## 2.6 RICE+ Backlog Branch

**Overview:**  
This branch creates a Notion page in a RICE+ backlog database and generates a synthetic routing reference for the original signal.

**Nodes Involved:**
- Create RICE+ Entry
- Parse RICE Response

### Node Details

#### Create RICE+ Entry
- **Type and technical role:** Notion node creating a database page
- **Configuration choices:**
  - Resource: `databasePage`
  - Database ID: `31e06bab-3ebe-81fe-9d0c-e65a49a99ef3`
  - Title: `[Signal] {{$json.signal}}`
  - Sets properties:
    - `Source|select` = `Signal Catcher`
    - `Status|select` = `Proposed`
    - `Customer Request|rich_text` left configured but no explicit mapped value shown
- **Key expressions or variables used:**
  - `{{ '[Signal] ' + $json.signal }}`
- **Input and output connections:**
  - Input from **Route Destination**
  - Output to **Parse RICE Response**
- **Version-specific requirements:** Notion node `typeVersion 2.2`
- **Edge cases or potential failure types:**
  - Database schema mismatch for property names/types
  - If select options do not exist, Notion may reject the write
  - `Customer Request|rich_text` appears present but effectively unset, which may be unintended
- **Sub-workflow reference:** None

#### Parse RICE Response
- **Type and technical role:** Code node normalizing Notion response
- **Configuration choices:**
  - Generates pseudo-reference `RICE-###`
  - Reads original signal from **Read Signal Details**
  - Returns:
    - merged original signal
    - `routedReference`
    - `routedUrl`
    - `notionRicePageId`
- **Key expressions or variables used:**
  - `Math.random()`
  - `$('Read Signal Details').first().json`
  - `$json.url`
  - `$json.id`
- **Input and output connections:**
  - Input from **Create RICE+ Entry**
  - Output to **Update Signal Status**
- **Version-specific requirements:** Code node `typeVersion 2`
- **Edge cases or potential failure types:**
  - Random reference is not guaranteed unique
  - `url` or `id` may be absent depending on API response shape
  - Synthetic reference is not tied to actual Notion page numbering
- **Sub-workflow reference:** None

---

## 2.7 Customer Health Branch

**Overview:**  
This branch creates a Customer Health entry in Notion and generates a readable health reference based on the customer name when available.

**Nodes Involved:**
- Create Customer Health Entry
- Parse Customer Health Response

### Node Details

#### Create Customer Health Entry
- **Type and technical role:** Notion node creating a database page
- **Configuration choices:**
  - Resource: `databasePage`
  - Database ID: `31e06bab-3ebe-811b-b204-c5f41b273303`
  - Title: `[Signal] {{$json.signal}}`
  - Sets:
    - `Account|rich_text`
    - `Type|select` based on signal type:
      - Customer Escalation → Escalation
      - Bug Report → Support Issue
      - else → Product Signal
    - `Sentiment|select` = `{{$json.sentiment}}`
    - `Date|date`
    - `Notes|rich_text`
- **Key expressions or variables used:**
  - `{{ '[Signal] ' + $json.signal }}`
  - `{{$json.signalType === 'Customer Escalation' ? 'Escalation' : $json.signalType === 'Bug Report' ? 'Support Issue' : 'Product Signal' }}`
  - `{{$json.sentiment}}`
- **Input and output connections:**
  - Input from **Route Destination**
  - Output to **Parse Customer Health Response**
- **Version-specific requirements:** Notion node `typeVersion 2.2`
- **Edge cases or potential failure types:**
  - This node writes into database ID `31e06bab-3ebe-811b-b204-c5f41b273303`, which is the same ID used by the trigger database; that may be intentional or may create routing feedback loops if schema overlaps
  - Several properties appear configured without visible mapped values, which may result in blank fields
  - Select option names must exist in Notion
- **Sub-workflow reference:** None

#### Parse Customer Health Response
- **Type and technical role:** Code node normalizing Customer Health page creation response
- **Configuration choices:**
  - Builds a synthetic reference from customer name:
    - first 3 non-space uppercase characters + `-HEALTH-###`
    - or `UNK-HEALTH-###` if no customer
  - Returns merged signal plus:
    - `routedReference`
    - `routedUrl`
    - `notionHealthPageId`
- **Key expressions or variables used:**
  - `signalData.customerMentioned`
  - `replace(/\s+/g, '')`
  - `substring(0, 3).toUpperCase()`
  - `Math.random()`
  - `$json.url`
  - `$json.id`
- **Input and output connections:**
  - Input from **Create Customer Health Entry**
  - Output to **Update Signal Status**
- **Version-specific requirements:** Code node `typeVersion 2`
- **Edge cases or potential failure types:**
  - Random reference is not unique-safe
  - Customer names shorter than 3 characters still work but produce short prefixes
  - If writing into the same source database, later polling logic could pick up derived entries depending on schema and status values
- **Sub-workflow reference:** None

---

## 2.8 Sprint Backlog Branch

**Overview:**  
This branch creates a backlog item in Notion for the active sprint and generates a sprint-style synthetic reference.

**Nodes Involved:**
- Create Sprint Backlog Item
- Parse Sprint Response

### Node Details

#### Create Sprint Backlog Item
- **Type and technical role:** Notion node creating a database page
- **Configuration choices:**
  - Resource: `databasePage`
  - Database ID: `31e06bab-3ebe-81fe-9d0c-e65a49a99ef3`
  - Title: `[Signal] {{$json.signal}}`
  - Sets:
    - `Status|select` = `To Do`
    - `Priority|select` = `{{$json.urgency}}`
    - `Source|select` = `Signal Catcher`
    - `Sprint|select` = `Sprint 24`
    - `Description|rich_text`
- **Key expressions or variables used:**
  - `{{ '[Signal] ' + $json.signal }}`
  - `{{$json.urgency}}`
- **Input and output connections:**
  - Input from **Route Destination**
  - Output to **Parse Sprint Response**
- **Version-specific requirements:** Notion node `typeVersion 2.2`
- **Edge cases or potential failure types:**
  - Hardcoded sprint value `Sprint 24` will become outdated
  - Priority select value must exist in Notion and match urgency labels exactly
  - `Description|rich_text` appears configured without a mapped value
- **Sub-workflow reference:** None

#### Parse Sprint Response
- **Type and technical role:** Code node normalizing sprint item creation response
- **Configuration choices:**
  - Generates pseudo-reference `SPRINT-###`
  - Merges original signal data
  - Adds:
    - `routedReference`
    - `routedUrl`
    - `notionSprintPageId`
- **Key expressions or variables used:**
  - `Math.random()`
  - `$('Read Signal Details').first().json`
  - `$json.url`
  - `$json.id`
- **Input and output connections:**
  - Input from **Create Sprint Backlog Item**
  - Output to **Update Signal Status**
- **Version-specific requirements:** Code node `typeVersion 2`
- **Edge cases or potential failure types:**
  - Same uniqueness caveat as other synthetic references
- **Sub-workflow reference:** None

---

## 2.9 Closed Loop Update and Slack Confirmation

**Overview:**  
This block finalizes the process by updating the original Notion signal to show it has been routed and then replies in Slack so stakeholders can see where the signal went.

**Nodes Involved:**
- Update Signal Status
- Build Thread Reply
- Reply in Thread

### Node Details

#### Update Signal Status
- **Type and technical role:** Notion node updating the original database page
- **Configuration choices:**
  - Resource: `databasePage`
  - Operation: `update`
  - Page ID: `{{$json.notionPageId}}`
  - Sets:
    - `Route Status|select` = `Routed`
    - `Routed Reference|rich_text`
  - The routed reference field is configured but no explicit expression is shown in the export, so its actual value mapping should be verified in n8n
- **Key expressions or variables used:**
  - `{{$json.notionPageId}}`
- **Input and output connections:**
  - Inputs from:
    - **Parse Jira Bug Response**
    - **Parse Jira Feature Response**
    - **Parse RICE Response**
    - **Parse Customer Health Response**
    - **Parse Sprint Response**
  - Output to **Build Thread Reply**
- **Version-specific requirements:** Notion node `typeVersion 2.2`
- **Edge cases or potential failure types:**
  - If `notionPageId` is missing, the update will fail
  - Property type/schema mismatch on `Route Status` or `Routed Reference`
  - If `Routed Reference` is not actually mapped, the record may show Routed without useful traceability
- **Sub-workflow reference:** None

#### Build Thread Reply
- **Type and technical role:** Code node creating Slack-ready reply metadata
- **Configuration choices:**
  - Maps source channel names to hardcoded Slack channel IDs:
    - `#product` → `C01PRODUCT`
    - `#support` → `C02SUPPORT`
    - `#sales` → `C03SALES`
    - `#engineering` → `C04ENGINEERING`
    - `#general` → `C05GENERAL`
  - Falls back to `signal.sourceChannel` if no mapping exists
  - Maps route destination to a human-readable label
  - Builds reply:
    - `Captured and routed to *<reference>* (<destinationLabel>)`
  - Outputs:
    - `slackChannelId`
    - `replyMessage`
- **Key expressions or variables used:**
  - `signal.sourceChannel`
  - `signal.routeDestination`
  - `signal.routedReference`
- **Input and output connections:**
  - Input from **Update Signal Status**
  - Output to **Reply in Thread**
- **Version-specific requirements:** Code node `typeVersion 2`
- **Edge cases or potential failure types:**
  - Hardcoded channel map must match the real Slack workspace
  - If source channel is stored in an unexpected format, Slack post may fail
  - Route destination labels are also hardcoded and can drift from upstream values
- **Sub-workflow reference:** None

#### Reply in Thread
- **Type and technical role:** HTTP Request node posting a Slack threaded message
- **Configuration choices:**
  - Method: `POST`
  - URL: `https://slack.com/api/chat.postMessage`
  - Uses predefined Slack API credential
  - Sends JSON body:
    - `channel: $json.slackChannelId`
    - `text: $json.replyMessage`
    - `thread_ts: $json.slackMessageId`
    - `unfurl_links: false`
  - Header `Content-Type: application/json`
  - `continueOnFail: true`
- **Key expressions or variables used:**
  - `$json.slackChannelId`
  - `$json.replyMessage`
  - `$json.slackMessageId`
- **Input and output connections:**
  - Input from **Build Thread Reply**
  - No downstream node
- **Version-specific requirements:** HTTP Request node `typeVersion 4.2`
- **Edge cases or potential failure types:**
  - Slack credential missing scope for `chat.postMessage`
  - Invalid channel ID or thread timestamp
  - If `slackMessageId` is not a valid Slack thread timestamp, the message may post as a normal message or fail
  - Because `continueOnFail` is enabled, Slack posting errors will not stop the workflow
- **Sub-workflow reference:** None

---

## 2.10 Documentation Sticky Notes

**Overview:**  
These nodes are non-executable annotations embedded in the canvas. They provide context about the workflow logic and should be preserved when rebuilding for maintainability.

**Nodes Involved:**
- Sticky Note - How It Works
- Sticky Note - Routing
- Sticky Note

### Node Details

#### Sticky Note - How It Works
- **Type and technical role:** Sticky Note, documentation-only
- **Configuration choices:**
  - Content explains that changing Route Status to `Routing` in Notion triggers the workflow and routes the signal to one of several destinations
- **Input and output connections:** None
- **Version-specific requirements:** Sticky Note `typeVersion 1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

#### Sticky Note - Routing
- **Type and technical role:** Sticky Note, documentation-only
- **Configuration choices:**
  - Content labels the routing engine and explains that signal details are flattened before a 5-way switch on Route Destination
- **Input and output connections:** None
- **Version-specific requirements:** Sticky Note `typeVersion 1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

#### Sticky Note
- **Type and technical role:** Sticky Note, documentation-only
- **Configuration choices:**
  - Content explains the closed-loop behavior: update status, add reference, and reply in Slack
- **Input and output connections:** None
- **Version-specific requirements:** Sticky Note `typeVersion 1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note - How It Works | Sticky Note | Canvas documentation describing overall purpose |  |  | ## How it works\nWhen you set a signal's Route Status to 'Routing' in Notion, this workflow picks it up and sends it to the right destination: Jira bug, Jira feature, RICE+ backlog, customer health, or sprint backlog. |
| Sticky Note - Routing | Sticky Note | Canvas documentation for routing engine |  |  | ## Routing Engine\nReads signal details from Notion, flattens properties, then routes via a 5-way switch based on the Route Destination field you selected. |
| Signal Route Trigger | Notion | Poll source Notion database for signal pages |  | Filter Routing Status | ## Routing Engine\nReads signal details from Notion, flattens properties, then routes via a 5-way switch based on the Route Destination field you selected. |
| Filter Routing Status | IF | Allow only records marked Routing | Signal Route Trigger | Read Signal Details | ## Routing Engine\nReads signal details from Notion, flattens properties, then routes via a 5-way switch based on the Route Destination field you selected. |
| Read Signal Details | Code | Flatten Notion properties into reusable JSON | Filter Routing Status | Route Destination | ## Routing Engine\nReads signal details from Notion, flattens properties, then routes via a 5-way switch based on the Route Destination field you selected. |
| Route Destination | Switch | Branch by routing destination | Read Signal Details | Create Jira Bug; Create Jira Feature; Create RICE+ Entry; Create Customer Health Entry; Create Sprint Backlog Item | ## Routing Engine\nReads signal details from Notion, flattens properties, then routes via a 5-way switch based on the Route Destination field you selected. |
| Create Jira Bug | HTTP Request | Create Jira bug issue | Route Destination | Parse Jira Bug Response | ## Routing Engine\nReads signal details from Notion, flattens properties, then routes via a 5-way switch based on the Route Destination field you selected. |
| Parse Jira Bug Response | Code | Normalize Jira bug response | Create Jira Bug | Update Signal Status | ## Routing Engine\nReads signal details from Notion, flattens properties, then routes via a 5-way switch based on the Route Destination field you selected. |
| Create Jira Feature | HTTP Request | Create Jira story for feature signal | Route Destination | Parse Jira Feature Response | ## Routing Engine\nReads signal details from Notion, flattens properties, then routes via a 5-way switch based on the Route Destination field you selected. |
| Parse Jira Feature Response | Code | Normalize Jira feature response | Create Jira Feature | Update Signal Status | ## Routing Engine\nReads signal details from Notion, flattens properties, then routes via a 5-way switch based on the Route Destination field you selected. |
| Create RICE+ Entry | Notion | Create feature backlog record in Notion | Route Destination | Parse RICE Response | ## Routing Engine\nReads signal details from Notion, flattens properties, then routes via a 5-way switch based on the Route Destination field you selected. |
| Parse RICE Response | Code | Normalize RICE backlog response | Create RICE+ Entry | Update Signal Status | ## Routing Engine\nReads signal details from Notion, flattens properties, then routes via a 5-way switch based on the Route Destination field you selected. |
| Create Customer Health Entry | Notion | Create customer health record in Notion | Route Destination | Parse Customer Health Response | ## Routing Engine\nReads signal details from Notion, flattens properties, then routes via a 5-way switch based on the Route Destination field you selected. |
| Parse Customer Health Response | Code | Normalize customer health response | Create Customer Health Entry | Update Signal Status | ## Routing Engine\nReads signal details from Notion, flattens properties, then routes via a 5-way switch based on the Route Destination field you selected. |
| Create Sprint Backlog Item | Notion | Create sprint backlog item in Notion | Route Destination | Parse Sprint Response | ## Routing Engine\nReads signal details from Notion, flattens properties, then routes via a 5-way switch based on the Route Destination field you selected. |
| Parse Sprint Response | Code | Normalize sprint backlog response | Create Sprint Backlog Item | Update Signal Status | ## Routing Engine\nReads signal details from Notion, flattens properties, then routes via a 5-way switch based on the Route Destination field you selected. |
| Update Signal Status | Notion | Mark original signal as routed | Parse Jira Bug Response; Parse Jira Feature Response; Parse RICE Response; Parse Customer Health Response; Parse Sprint Response | Build Thread Reply | ## Closed Loop\nAfter routing, updates the signal status to 'Routed' with a reference link, and replies in the original Slack thread to confirm where it went. |
| Build Thread Reply | Code | Prepare Slack channel ID and confirmation text | Update Signal Status | Reply in Thread | ## Closed Loop\nAfter routing, updates the signal status to 'Routed' with a reference link, and replies in the original Slack thread to confirm where it went. |
| Reply in Thread | HTTP Request | Post confirmation reply to Slack thread | Build Thread Reply |  | ## Closed Loop\nAfter routing, updates the signal status to 'Routed' with a reference link, and replies in the original Slack thread to confirm where it went. |
| Sticky Note | Sticky Note | Canvas documentation for closing sequence |  |  | ## Closed Loop\nAfter routing, updates the signal status to 'Routed' with a reference link, and replies in the original Slack thread to confirm where it went. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it **Signal Router**.
   - Keep the workflow inactive until credentials and mappings are verified.

2. **Add a Sticky Note for the workflow purpose**
   - Create a Sticky Note node.
   - Title/content:
     - `## How it works`
     - `When you set a signal's Route Status to 'Routing' in Notion, this workflow picks it up and sends it to the right destination: Jira bug, Jira feature, RICE+ backlog, customer health, or sprint backlog.`

3. **Add the Notion trigger/query node**
   - Add a **Notion** node named **Signal Route Trigger**.
   - Configure:
     - Resource: `Database Page`
     - Operation: `Get Many` / `Get All`
     - Database ID: `31e06bab-3ebe-811b-b204-c5f41b273303`
     - Limit: `10`
     - Simple: `false`
   - Enable polling on the node.
   - Attach a valid **Notion credential** with access to the source database.

4. **Add the status filter**
   - Add an **IF** node named **Filter Routing Status**.
   - Condition:
     - Left value: `={{ $json.properties?.['Route Status']?.select?.name }}`
     - Operator: `equals`
     - Right value: `Routing`
   - Connect **Signal Route Trigger → Filter Routing Status**.
   - Use the **true** output only.

5. **Add the signal-flattening code node**
   - Add a **Code** node named **Read Signal Details**.
   - Paste this logic conceptually:
     - Read `properties` from the Notion page
     - Extract plain values from title, rich text, select, url, and date types
     - Return a flat JSON object
   - Include these output fields:
     - `notionPageId`
     - `signal`
     - `dateCaptured`
     - `sourceChannel`
     - `originalMessage`
     - `author`
     - `signalType`
     - `sentiment`
     - `urgency`
     - `aiAnalysis`
     - `customerMentioned`
     - `routeDestination`
     - `routeStatus`
     - `threadUrl`
     - `slackMessageId`
   - Connect **Filter Routing Status (true) → Read Signal Details**.

6. **Add a routing Sticky Note**
   - Create a Sticky Note with:
     - `## Routing Engine`
     - `Reads signal details from Notion, flattens properties, then routes via a 5-way switch based on the Route Destination field you selected.`

7. **Add the Switch node**
   - Add a **Switch** node named **Route Destination**.
   - Set five rules on `={{ $json.routeDestination }}`:
     1. equals `Jira Bug`
     2. equals `Jira Feature`
     3. equals `RICE+ Backlog`
     4. equals `Customer Health`
     5. equals `Sprint Backlog`
   - Connect **Read Signal Details → Route Destination**.

---

## Jira Bug branch

8. **Create the Jira Bug HTTP node**
   - Add an **HTTP Request** node named **Create Jira Bug**.
   - Configure:
     - Method: `POST`
     - URL: `https://company.atlassian.net/rest/api/3/issue`
     - Authentication: **HTTP Basic Auth** credential
     - Send body as JSON
     - Header: `Content-Type: application/json`
   - Build a Jira issue payload with:
     - Project key `PROJ`
     - Issue type `Bug`
     - Summary `[Signal] {{$json.signal}}`
     - ADF description containing source, original message, AI analysis, and Slack thread link
     - Priority mapping from urgency
     - Labels `signal-catcher`, `slack-signal`
   - Connect **Route Destination output 0 → Create Jira Bug**.
   - Credential requirement:
     - Jira user email + API token via HTTP Basic Auth.

9. **Add bug response parser**
   - Add a **Code** node named **Parse Jira Bug Response**.
   - Logic:
     - Read Jira key from current item
     - Read original data from `$('Read Signal Details').first().json`
     - Return merged payload with:
       - `routedReference`
       - `routedUrl`
   - Connect **Create Jira Bug → Parse Jira Bug Response**.

---

## Jira Feature branch

10. **Create the Jira Feature HTTP node**
    - Add an **HTTP Request** node named **Create Jira Feature**.
    - Same Jira connection settings as the bug branch.
    - Payload differences:
      - Issue type `Story`
      - Include customer mentioned in the description
      - Labels `signal-catcher`, `feature-signal`, `slack-signal`
      - Priority mapping:
        - Critical → Highest
        - High → High
        - else → Medium
    - Connect **Route Destination output 1 → Create Jira Feature**.

11. **Add feature response parser**
    - Add a **Code** node named **Parse Jira Feature Response**.
    - Same normalization pattern as the bug parser.
    - Connect **Create Jira Feature → Parse Jira Feature Response**.

---

## RICE+ backlog branch

12. **Create the RICE+ Notion node**
    - Add a **Notion** node named **Create RICE+ Entry**.
    - Configure:
      - Resource: `Database Page`
      - Operation: `Create`
      - Database ID: `31e06bab-3ebe-81fe-9d0c-e65a49a99ef3`
      - Title: `={{ '[Signal] ' + $json.signal }}`
    - Set properties:
      - `Source` select = `Signal Catcher`
      - `Status` select = `Proposed`
      - `Customer Request` rich text as needed
    - Connect **Route Destination output 2 → Create RICE+ Entry**.
    - Use a Notion credential with write access to that database.

13. **Add RICE response parser**
    - Add a **Code** node named **Parse RICE Response**.
    - Logic:
      - Generate a readable reference like `RICE-123`
      - Merge original signal data from **Read Signal Details**
      - Include:
        - `routedReference`
        - `routedUrl`
        - `notionRicePageId`
    - Connect **Create RICE+ Entry → Parse RICE Response**.

---

## Customer Health branch

14. **Create the Customer Health Notion node**
    - Add a **Notion** node named **Create Customer Health Entry**.
    - Configure:
      - Resource: `Database Page`
      - Operation: `Create`
      - Database ID: `31e06bab-3ebe-811b-b204-c5f41b273303`
      - Title: `={{ '[Signal] ' + $json.signal }}`
    - Set properties:
      - `Account` rich text
      - `Type` select:
        - `Escalation` when signal type is `Customer Escalation`
        - `Support Issue` when signal type is `Bug Report`
        - else `Product Signal`
      - `Sentiment` select = `={{ $json.sentiment }}`
      - `Date` date
      - `Notes` rich text
    - Connect **Route Destination output 3 → Create Customer Health Entry**.
    - Verify carefully whether this should really target the same database as the source trigger.

15. **Add customer health parser**
    - Add a **Code** node named **Parse Customer Health Response**.
    - Logic:
      - Build a reference from customer name if present, otherwise `UNK-HEALTH-###`
      - Merge original signal data
      - Add:
        - `routedReference`
        - `routedUrl`
        - `notionHealthPageId`
    - Connect **Create Customer Health Entry → Parse Customer Health Response**.

---

## Sprint backlog branch

16. **Create the Sprint Backlog Notion node**
    - Add a **Notion** node named **Create Sprint Backlog Item**.
    - Configure:
      - Resource: `Database Page`
      - Operation: `Create`
      - Database ID: `31e06bab-3ebe-81fe-9d0c-e65a49a99ef3`
      - Title: `={{ '[Signal] ' + $json.signal }}`
    - Set properties:
      - `Status` select = `To Do`
      - `Priority` select = `={{ $json.urgency }}`
      - `Source` select = `Signal Catcher`
      - `Sprint` select = `Sprint 24`
      - `Description` rich text
    - Connect **Route Destination output 4 → Create Sprint Backlog Item**.

17. **Add sprint response parser**
    - Add a **Code** node named **Parse Sprint Response**.
    - Logic:
      - Generate reference like `SPRINT-123`
      - Merge original signal data
      - Add:
        - `routedReference`
        - `routedUrl`
        - `notionSprintPageId`
    - Connect **Create Sprint Backlog Item → Parse Sprint Response**.

---

## Common closing sequence

18. **Add the status update node**
    - Add a **Notion** node named **Update Signal Status**.
    - Configure:
      - Resource: `Database Page`
      - Operation: `Update`
      - Page ID: `={{ $json.notionPageId }}`
    - Set properties:
      - `Route Status` select = `Routed`
      - `Routed Reference` rich text = routed reference from previous branch
    - Connect all five parser nodes into this node:
      - **Parse Jira Bug Response**
      - **Parse Jira Feature Response**
      - **Parse RICE Response**
      - **Parse Customer Health Response**
      - **Parse Sprint Response**
    - Important:
      - In the exported workflow, the field configuration for `Routed Reference` appears incomplete. When rebuilding, explicitly map it to `={{ $json.routedReference }}` or equivalent.

19. **Add a closed-loop Sticky Note**
    - Create a Sticky Note with:
      - `## Closed Loop`
      - `After routing, updates the signal status to 'Routed' with a reference link, and replies in the original Slack thread to confirm where it went.`

20. **Add the Slack message builder**
    - Add a **Code** node named **Build Thread Reply**.
    - Logic:
      - Map source channel names to Slack channel IDs
      - Build destination labels:
        - Jira Bug → Jira Bug Ticket
        - Jira Feature → Jira Feature Story
        - RICE+ Backlog → RICE+ Feature Backlog
        - Customer Health → Customer Health Tracker
        - Sprint Backlog → Sprint 24 Backlog
      - Compose:
        - `Captured and routed to *${signal.routedReference}* (${destinationLabel})`
      - Output:
        - `slackChannelId`
        - `replyMessage`
    - Connect **Update Signal Status → Build Thread Reply**.

21. **Add the Slack post node**
    - Add an **HTTP Request** node named **Reply in Thread**.
    - Configure:
      - Method: `POST`
      - URL: `https://slack.com/api/chat.postMessage`
      - Authentication: **Slack API predefined credential**
      - Body as JSON
      - Header: `Content-Type: application/json`
      - Body fields:
        - `channel: {{$json.slackChannelId}}`
        - `text: {{$json.replyMessage}}`
        - `thread_ts: {{$json.slackMessageId}}`
        - `unfurl_links: false`
      - Enable **Continue On Fail**
    - Connect **Build Thread Reply → Reply in Thread**.
    - Credential requirement:
      - Slack app token with permission to post messages in the target channels, typically including `chat:write`.

22. **Validate all Notion schemas**
    - Confirm property names exactly match:
      - Source signal database:
        - `Signal`
        - `Date Captured`
        - `Source Channel`
        - `Original Message`
        - `Author`
        - `Signal Type`
        - `Sentiment`
        - `Urgency`
        - `AI Analysis`
        - `Customer Mentioned`
        - `Route Destination`
        - `Route Status`
        - `Thread URL`
        - `Slack Message ID`
        - `Routed Reference`
      - Destination databases:
        - RICE+ properties
        - Customer Health properties
        - Sprint properties

23. **Validate all external credentials**
    - **Notion**
      - Integration must have read/write access to all referenced databases
    - **Jira**
      - HTTP Basic Auth with Atlassian email + API token
      - Account must be allowed to create issues in project `PROJ`
    - **Slack**
      - Credential must authorize `chat.postMessage`
      - Bot must be present in each destination channel if required by workspace settings

24. **Test each route manually**
    - In the source Notion database, create or update one record per route destination.
    - Set:
      - `Route Status = Routing`
      - `Route Destination` to each of the five expected values
    - Confirm:
      - Correct branch runs
      - Routed asset is created
      - Source page is updated to `Routed`
      - Slack reply appears in the correct thread

25. **Review hardcoded values before production use**
    - Replace placeholder Jira hostname `company.atlassian.net`
    - Replace project key `PROJ`
    - Replace hardcoded Slack channel IDs
    - Replace hardcoded sprint label `Sprint 24`
    - Consider adding a fallback branch for unknown route destinations
    - Consider storing actual destination URLs in Notion, not just generated references

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| This workflow is currently inactive in the provided export. | Activate only after credentials, database schemas, and mappings are verified. |
| The workflow relies heavily on exact Notion property names. Renaming fields in Notion will break extraction and updates. | Applies across the entire workflow. |
| Several Notion create/update nodes show properties configured without clearly visible mapped values in the export. | Re-check field mappings manually in n8n before deployment. |
| The Customer Health branch appears to write into the same Notion database ID used by the source trigger. | Verify whether this is intentional to avoid recursive or confusing records. |
| Random references such as `RICE-###`, `SPRINT-###`, and `XXX-HEALTH-###` are human-friendly but not guaranteed unique. | Consider replacing with page IDs, timestamps, or deterministic identifiers. |
| No fallback route exists if `Route Destination` contains an unexpected value. | Consider adding a default error-handling branch. |
| Slack posting is non-blocking because `Reply in Thread` uses Continue On Fail. | Routing can complete even if Slack confirmation fails. |