Real estate property matching & lead management with Airtable, Gmail and Slack

https://n8nworkflows.xyz/workflows/real-estate-property-matching---lead-management-with-airtable--gmail-and-slack-11914


# Real estate property matching & lead management with Airtable, Gmail and Slack

## Disclaimer (provided)
Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

# 1. Workflow Overview

**Title:** Real estate property matching & lead management with Airtable, Gmail and Slack

**Purpose:**  
This workflow captures inbound real-estate buyer leads via a webhook, searches an Airtable “Properties” database for matching listings (including a ±5% budget tolerance), and then:
- If matches exist: generates a rich HTML email showing “property cards”, emails the lead via Gmail, logs the lead into an Airtable “Buyers” table, notifies the sales team in Slack, and returns a success response to the webhook request.
- If no matches: returns a “no properties found” response to the webhook request.

**Target use cases:**
- Website “property match” forms
- Lead routing and quick follow-up automation for sales teams
- Lightweight property recommendation engine powered by Airtable formulas

## Logical blocks
### 1.1 Lead Intake & Normalization
Receives POSTed lead data and normalizes key fields for later formula building.

### 1.2 Property Search Engine (Airtable Formula + Search)
Builds a dynamic Airtable `filterByFormula` using city, property type text matching, and budget tolerance, then searches the Properties table.

### 1.3 Match Decision
Checks whether the Airtable search returned at least one record.

### 1.4 Success Flow (Email + Logging + Notifications + Webhook Response)
Generates HTML email content from returned properties, sends it via Gmail, logs the lead into Airtable, alerts Slack, and responds to the webhook.

### 1.5 No-Match Response
Responds immediately to the webhook if there are no matching properties.

---

# 2. Block-by-Block Analysis

## 2.1 Lead Intake & Normalization

**Overview:**  
Captures lead data from an HTTP endpoint and ensures required fields are consistently placed under `body.*` so downstream nodes can rely on stable paths.

**Nodes involved:**
- Capture Lead
- Structure & Clean Data

### Node: Capture Lead
- **Type / role:** `Webhook` — entry point that receives lead submissions.
- **Configuration (interpreted):**
  - **Method:** `POST`
  - **Path:** `property-match` (your full URL depends on n8n base URL)
  - **Response mode:** `Respond to Webhook` node (the workflow must end with a Respond node on each branch to avoid timeouts).
- **Input/Output:**
  - **Input:** External HTTP POST payload.
  - **Output:** A JSON object containing at least `body` (the posted data).
- **Edge cases / failures:**
  - If the request does not include JSON (or is malformed), `body` may be empty or not parsed as expected.
  - If no Respond node is reached, the webhook caller can time out.
- **Version notes:** TypeVersion `2.1` (modern webhook node behavior; response handled by Respond node).

### Node: Structure & Clean Data
- **Type / role:** `Set` — normalizes/aliases fields.
- **Configuration choices:**
  - Sets:
    - `body.budget` = `{{$json.body.budget}}`
    - `body.city` = `{{$json.body.city}}`
    - `body.property_type` = `{{$json.body.type}}`
  - This effectively maps incoming `type` → `property_type` while keeping it inside `body`.
- **Key expressions / variables:**
  - Uses `$json.body.*` paths from the webhook.
- **Input/Output connections:**
  - **Input:** Capture Lead
  - **Output:** Formula Creation
- **Edge cases / failures:**
  - If `body` or `body.type` is missing, `body.property_type` becomes empty; later formula may become too broad or invalid.
- **Version notes:** TypeVersion `3.4`.

**Sticky note context (covers this section):**
- **“Lead Intake Section”**: “Captures incoming lead data via webhook and structures it for processing (budget, city, property type).”
- **“Sample Data & Resources”** includes sample payload and Airtable share link (see Section 5).

---

## 2.2 Property Search Engine (Formula + Airtable Search)

**Overview:**  
Builds an Airtable `filterByFormula` string that matches: (a) property title contains the requested property type (case-insensitive), (b) exact city match, and (c) price is within budget or within ±5% tolerance. Then executes an Airtable search.

**Nodes involved:**
- Formula Creation
- Fetch Properties Required & ± 5% budget range

### Node: Formula Creation
- **Type / role:** `Code` — constructs Airtable formula dynamically from lead data.
- **Configuration choices:**
  - Reads inputs from `items[0].json.body` (fallback: `items[0].json`)
  - Extracts:
    - `searchTerm` from `property_type`
    - `city` from `city` with quotes escaped
    - `budget` parsed from string by removing currency symbol `₹` and commas
  - Computes ±5% bounds:
    - `lowerLimit = floor(budget * 0.95)`
    - `upperLimit = ceil(budget * 1.05)`
  - Constructs Airtable formula (simplified meaning):
    - `Title` contains `searchTerm` (case-insensitive)
    - `Location` equals `city`
    - `Price` numeric value is either:
      - `<= budget`, OR
      - between `lowerLimit` and `upperLimit`
- **Key expressions / variables:**
  - Uses template string to embed `searchTerm`, `city`, `budget`, bounds.
  - Price conversion uses Airtable formula:
    - `VALUE(SUBSTITUTE(SUBSTITUTE({Price}, "₹", ""), ",", ""))`
- **Input/Output connections:**
  - **Input:** Structure & Clean Data
  - **Output:** Fetch Properties Required & ± 5% budget range (outputs `{ formula }`)
- **Edge cases / failures:**
  - If `budget` is missing or non-numeric, `parseInt` may return `NaN`, causing an invalid Airtable formula.
  - If `searchTerm` is empty, `FIND("", ...)` is always `1` in Airtable, which can unintentionally match all titles (depending on Airtable behavior). This can lead to overly broad results.
  - If `city` contains quotes or unusual characters, escaping is partial (only `"`), but other special characters could still affect formula.
- **Version notes:** TypeVersion `2`.

### Node: Fetch Properties Required & ± 5% budget range
- **Type / role:** `Airtable` — searches property listings using the generated formula.
- **Configuration choices:**
  - Operation: `search`
  - Base: `Property Matching(Real Estate)` (`appe77qnWMLUWfKEe`)
  - Table: `Properties` (`tblWdmd25jGWaqPd3`)
  - `filterByFormula` = `{{$json.formula}}`
  - **Always output data:** enabled (`alwaysOutputData: true`) so downstream nodes run even if zero records are found.
- **Input/Output connections:**
  - **Input:** Formula Creation
  - **Output:** Check Match Availability
- **Edge cases / failures:**
  - Airtable auth/permission issues (invalid token, base/table not shared).
  - Airtable rate limits or transient 429/5xx.
  - Formula syntax error (from NaN budget or unescaped text) will cause Airtable API error.
- **Version notes:** TypeVersion `2.1`.

**Sticky note context (covers this section):**
- **“Property Search Section”**: “Builds Airtable formula with ±5% budget flexibility, searches properties database, and validates if matches exist.”

---

## 2.3 Match Decision

**Overview:**  
Determines whether any Airtable record was returned by checking if the current item contains a non-empty `id` (Airtable record id).

**Nodes involved:**
- Check Match Availability

### Node: Check Match Availability
- **Type / role:** `IF` — branches workflow into match vs no-match.
- **Configuration choices:**
  - Condition: `{{$json.id}}` **is not empty**
  - If true: proceed to HTML generation + email
  - If false: respond “no properties found”
- **Input/Output connections:**
  - **Input:** Fetch Properties Required & ± 5% budget range
  - **True output:** Generate Email Template
  - **False output:** No Properties Found Respond
- **Edge cases / failures:**
  - If Airtable node outputs a single empty item (because `alwaysOutputData` is on), `$json.id` may be undefined and correctly routes to “false”.
  - If Airtable returns multiple records, the IF node evaluates per-item; the “true” branch will receive one item per property, which is intended for building the email list later.
- **Version notes:** TypeVersion `2.2`.

---

## 2.4 Success Flow (Email + Logging + Slack + Response)

**Overview:**  
Builds an HTML email listing all matched properties, sends it to the lead, then logs the lead in Airtable, notifies Slack, and returns a JSON response to the webhook caller.

**Nodes involved:**
- Generate Email Template
- Send Property Details
- Append Lead Data
- Notify Sales Agent
- Send Lead Confirmation Message

### Node: Generate Email Template
- **Type / role:** `Code` — renders a full HTML email with one “card” per matched property.
- **Configuration choices:**
  - Expects each incoming item to be a property record with fields like:
    - `Title`, `Location`, `Type`, `Price`, `Bedrooms`, `Bathrooms`, `Size (sqft)`
    - `Images` (array of objects with `.url`)
    - `Features` (array of strings)
    - Suggested metadata: `Suggested Buyer Type`, `Is Luxury Property?`
  - Builds:
    - A `generatePropertyCard(property)` function
    - A full HTML document and stores it as `{ html: emailHtml }`
  - Generates a “View More Details” link using:
    - `https://your-website.com/property/${property.ID || ""}`
- **Input/Output connections:**
  - **Input:** Check Match Availability (true branch), receiving **multiple items** (one per property)
  - **Output:** Send Property Details (single item containing the HTML)
- **Edge cases / failures:**
  - Property fields may not exist or may be named differently in Airtable; the email will show “-” or “N/A”.
  - `Images` may not be an array; `.map` would fail if it’s null/undefined unless Airtable always returns `[]`. (Code uses `(property.Images || [])`, which is safe.)
  - The link uses `property.ID`, but Airtable’s record id is typically `id` and a custom field might be needed; otherwise links may end up blank.
- **Version notes:** TypeVersion `2`.

### Node: Send Property Details
- **Type / role:** `Gmail` — sends the HTML email to the lead.
- **Configuration choices:**
  - **To:** `{{ $('Capture Lead').first().json.body.email }}`
  - **Subject:** `🏡 Properties Matching Your Requirements` (configured as an expression with a leading `=` in the JSON; it effectively resolves to a string)
  - **Message body:** `{{$json.html}}` (HTML produced by prior Code node)
  - Option: `appendAttribution: false`
- **Credentials:** Gmail OAuth2 credential required.
- **Input/Output connections:**
  - **Input:** Generate Email Template
  - **Outputs (fan-out):**
    - Append Lead Data
    - Notify Sales Agent
    - Send Lead Confirmation Message
- **Edge cases / failures:**
  - Gmail OAuth token expired / insufficient scopes.
  - Invalid recipient email.
  - If HTML is very large (many properties/images), Gmail may reject or clip content.
- **Version notes:** TypeVersion `2.1`.

### Node: Append Lead Data
- **Type / role:** `Airtable` — creates a new Buyer lead record for tracking.
- **Configuration choices:**
  - Operation: `create`
  - Base: `Property Matching(Real Estate)` (`appe77qnWMLUWfKEe`)
  - Table: `Buyers` (`tbljB7wf7ccn3YjkE`)
  - Field mapping:
    - City/Name/Email/Budget/Preferred Property Type from `Capture Lead` body
    - Status = `New`
    - Lead Source = `Other`
    - Matched Properties = `{{ $('Fetch Properties Required & ± 5% budget range').first().json.Title }}`
      - Note: only logs the **first** matched property title.
- **Input/Output connections:**
  - **Input:** Send Property Details
  - **Output:** none downstream
- **Edge cases / failures:**
  - Airtable schema mismatch (field names must match exactly: “Preferred Property Type”, “Matched Properties”, etc.).
  - “Matched Properties” storing only the first property may be insufficient; if the field is meant to store multiple, consider concatenation.
- **Version notes:** TypeVersion `2.1`.

### Node: Notify Sales Agent
- **Type / role:** `Slack` — posts a channel message alerting sales about the new lead.
- **Configuration choices:**
  - Posts formatted message containing lead details from `Capture Lead`:
    - name, phone, email, city, interest/type
  - Channel selection via `channelId` (`C090F70N52M`) with cached name `website-uptime`
- **Credentials:** Slack OAuth2/API credential required.
- **Input/Output connections:**
  - **Input:** Send Property Details
  - **Output:** none downstream
- **Edge cases / failures:**
  - Missing Slack permissions (chat:write) or wrong channel ID.
  - If `phone` is missing from payload, message shows blank.
- **Version notes:** TypeVersion `2.3`.

### Node: Send Lead Confirmation Message
- **Type / role:** `Respond to Webhook` — returns success JSON to the original HTTP caller.
- **Configuration choices:**
  - Respond with: JSON body:
    - `"Great news! We’ve found matching properties so Please check your email — we’ve sent the property details 📩"`
- **Input/Output connections:**
  - **Input:** Send Property Details
  - **Output:** ends execution for the webhook request
- **Edge cases / failures:**
  - If Gmail fails, this node won’t run (because it’s downstream of Gmail). The webhook caller would not get a response unless you add error handling.
- **Version notes:** TypeVersion `1.4`.

**Sticky note context (covers this section):**
- **“Success Response Section”**: “Generates HTML email with property cards, sends to lead, logs in Airtable, notifies sales team, and respond to webhook.”

---

## 2.5 No-Match Response

**Overview:**  
If Airtable returns no records, the workflow responds with a helpful message and does not attempt to email/log/notify.

**Nodes involved:**
- No Properties Found Respond

### Node: No Properties Found Respond
- **Type / role:** `Respond to Webhook` — returns “no results” JSON.
- **Configuration choices:**
  - JSON body:
    - `"Sorry, we couldn’t find any properties matching your exact requirements. Please try adjusting your budget or property preferences"`
- **Input/Output connections:**
  - **Input:** Check Match Availability (false branch)
  - **Output:** ends execution for the webhook request
- **Edge cases / failures:**
  - None typical; only depends on the workflow reaching this branch.
- **Version notes:** TypeVersion `1.4`.

---

## 2.6 Documentation / Notes Nodes (Sticky Notes)

**Overview:**  
Sticky notes provide embedded guidance, sample payload, and setup steps. They do not affect execution.

**Nodes involved:**
- Workflow Overview (Sticky Note)
- Sample Data & Resources (Sticky Note)
- Lead Intake Section (Sticky Note)
- Property Search Section (Sticky Note)
- Success Response Section (Sticky Note)

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Capture Lead | Webhook | Entry point: receives lead payload | — | Structure & Clean Data | ## Lead Intake\nCaptures incoming lead data via webhook and structures it for processing (budget, city, property type). |
| Structure & Clean Data | Set | Normalize/alias fields into `body.*` | Capture Lead | Formula Creation | ## Lead Intake\nCaptures incoming lead data via webhook and structures it for processing (budget, city, property type). |
| Formula Creation | Code | Build Airtable `filterByFormula` with ±5% budget logic | Structure & Clean Data | Fetch Properties Required & ± 5% budget range | ## Property Search Engine\nBuilds Airtable formula with ±5% budget flexibility, searches properties database, and validates if matches exist. |
| Fetch Properties Required & ± 5% budget range | Airtable | Search properties via filter formula | Formula Creation | Check Match Availability | ## Property Search Engine\nBuilds Airtable formula with ±5% budget flexibility, searches properties database, and validates if matches exist. |
| Check Match Availability | IF | Branch based on whether any property record exists (`id` not empty) | Fetch Properties Required & ± 5% budget range | Generate Email Template; No Properties Found Respond | ## Property Search Engine\nBuilds Airtable formula with ±5% budget flexibility, searches properties database, and validates if matches exist. |
| Generate Email Template | Code | Render HTML email with property cards | Check Match Availability (true) | Send Property Details | ## Success Flow\nGenerates HTML email with property cards, sends to lead, logs in Airtable, notifies sales team, and respond to webhook. |
| Send Property Details | Gmail | Send the HTML property email to lead | Generate Email Template | Append Lead Data; Notify Sales Agent; Send Lead Confirmation Message | ## Success Flow\nGenerates HTML email with property cards, sends to lead, logs in Airtable, notifies sales team, and respond to webhook. |
| Append Lead Data | Airtable | Create lead record in Buyers table | Send Property Details | — | ## Success Flow\nGenerates HTML email with property cards, sends to lead, logs in Airtable, notifies sales team, and respond to webhook. |
| Notify Sales Agent | Slack | Post lead alert to Slack channel | Send Property Details | — | ## Success Flow\nGenerates HTML email with property cards, sends to lead, logs in Airtable, notifies sales team, and respond to webhook. |
| Send Lead Confirmation Message | Respond to Webhook | Return success JSON to caller | Send Property Details | — | ## Success Flow\nGenerates HTML email with property cards, sends to lead, logs in Airtable, notifies sales team, and respond to webhook. |
| No Properties Found Respond | Respond to Webhook | Return “no match” JSON to caller | Check Match Availability (false) | — | ## Property Search Engine\nBuilds Airtable formula with ±5% budget flexibility, searches properties database, and validates if matches exist. |
| Sticky Note | Sticky Note | Sample payload + Airtable resource link | — | — | ## Sample Data & Resources\nhttps://airtable.com/appe77qnWMLUWfKEe/shrQ9t6tboCuNkswq\n\n### Webhook Payload:\n\n{\n    "name":"Suresh",\n    "email":"johndeo@gmail.com",\n    "city": "Delhi",\n    "phone":"9836462538",\n    "type": "3BHK Apartment",\n    "budget": "4300000" \n} |
| Workflow Overview | Sticky Note | High-level explanation + setup steps | — | — | ## How it works\n1. Captures and structures the lead data\n2. Builds a dynamic Airtable formula to search properties matching their criteria (exact budget or ±5% range)\n3. Fetches matching properties from Airtable\n4. If matches found: generates a beautiful HTML email with property cards and sends it to the lead\n5. Logs the lead in Airtable and notifies the sales team via Slack\n6. Returns appropriate confirmation messages\n\n## Setup steps\n1. **Airtable**: Connect your Airtable account \n2. **Gmail**: Authenticate Gmail OAuth2 \n3. **Slack**: Connect Slack OAuth2 |
| Lead Intake Section | Sticky Note | Section label | — | — | ## Lead Intake\nCaptures incoming lead data via webhook and structures it for processing (budget, city, property type). |
| Property Search Section | Sticky Note | Section label | — | — | ## Property Search Engine\nBuilds Airtable formula with ±5% budget flexibility, searches properties database, and validates if matches exist. |
| Success Response Section | Sticky Note | Section label | — | — | ## Success Flow\nGenerates HTML email with property cards, sends to lead, logs in Airtable, notifies sales team, and respond to webhook. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it:  
   **Real estate property matching & lead management with Airtable, Gmail and Slack**

2. **Add Webhook node**: `Capture Lead`
   - Type: **Webhook**
   - HTTP Method: **POST**
   - Path: **property-match**
   - Response Mode: **Using “Respond to Webhook” node**
   - Save and note the test/production webhook URL.

3. **Add Set node**: `Structure & Clean Data`
   - Type: **Set**
   - Add fields (as expressions):
     - `body.budget` → `{{$json.body.budget}}`
     - `body.city` → `{{$json.body.city}}`
     - `body.property_type` → `{{$json.body.type}}`
   - Connect: `Capture Lead` → `Structure & Clean Data`

4. **Add Code node**: `Formula Creation`
   - Type: **Code**
   - Paste logic that:
     - Reads `property_type`, `city`, `budget`
     - Parses budget number
     - Builds `AND(FIND(...), {Location}="...", OR(price<=budget, AND(price between ±5%)))`
     - Outputs `{ formula: "..." }`
   - Connect: `Structure & Clean Data` → `Formula Creation`

5. **Add Airtable node**: `Fetch Properties Required & ± 5% budget range`
   - Type: **Airtable**
   - Credentials: connect Airtable (Personal Access Token or OAuth, depending on your n8n setup)
   - Operation: **Search**
   - Base: **Property Matching(Real Estate)** (or your base)
   - Table: **Properties**
   - Filter By Formula: `{{$json.formula}}`
   - Enable **Always Output Data** (so you can branch on “no results” cleanly)
   - Connect: `Formula Creation` → this Airtable node

6. **Add IF node**: `Check Match Availability`
   - Type: **IF**
   - Condition: **String → is not empty**
     - Left value: `{{$json.id}}`
   - Connect: `Fetch Properties...` → `Check Match Availability`

7. **Add Code node** (true branch): `Generate Email Template`
   - Type: **Code**
   - Build HTML using all incoming items (each is a property)
   - Output a single item with `{ html: "<!DOCTYPE html>..." }`
   - Connect (true output): `Check Match Availability` → `Generate Email Template`

8. **Add Gmail node**: `Send Property Details`
   - Type: **Gmail**
   - Credentials: connect **Gmail OAuth2** (ensure scope allows sending email)
   - Operation: **Send**
   - To: `{{ $('Capture Lead').first().json.body.email }}`
   - Subject: `🏡 Properties Matching Your Requirements`
   - Message: `{{$json.html}}`
   - Disable attribution if desired
   - Connect: `Generate Email Template` → `Send Property Details`

9. **Add Airtable node**: `Append Lead Data`
   - Type: **Airtable**
   - Credentials: same Airtable credential
   - Operation: **Create**
   - Base: **Property Matching(Real Estate)**
   - Table: **Buyers**
   - Map fields:
     - Name: `{{ $('Capture Lead').first().json.body.name }}`
     - Email: `{{ $('Capture Lead').first().json.body.email }}`
     - City: `{{ $('Capture Lead').first().json.body.city }}`
     - Budget: `{{ $('Capture Lead').first().json.body.budget }}`
     - Preferred Property Type: `{{ $('Capture Lead').first().json.body.type }}`
     - Status: `New`
     - Lead Source: `Other`
     - Matched Properties: `{{ $('Fetch Properties Required & ± 5% budget range').first().json.Title }}`
   - Connect: `Send Property Details` → `Append Lead Data`

10. **Add Slack node**: `Notify Sales Agent`
    - Type: **Slack**
    - Credentials: connect Slack OAuth2/API (must allow posting)
    - Operation: **Post message**
    - Channel: select your channel (ID like `C090F70N52M`)
    - Message text: interpolate lead fields from `Capture Lead` (name, phone, email, city, type)
    - Connect: `Send Property Details` → `Notify Sales Agent`

11. **Add Respond to Webhook node** (success): `Send Lead Confirmation Message`
    - Type: **Respond to Webhook**
    - Respond With: **JSON**
    - Body: success message indicating email was sent
    - Connect: `Send Property Details` → `Send Lead Confirmation Message`

12. **Add Respond to Webhook node** (no-match): `No Properties Found Respond`
    - Type: **Respond to Webhook**
    - Respond With: **JSON**
    - Body: “no properties found” message
    - Connect (false output): `Check Match Availability` → `No Properties Found Respond`

13. **Credentials checklist**
    - **Airtable:** configure base/table access; ensure field names match exactly.
    - **Gmail OAuth2:** ensure the connected account can send mail; re-auth if expired.
    - **Slack:** ensure bot/user has access to the target channel and `chat:write`.

14. **Test with sample webhook payload** (from sticky note)
    - POST JSON to the webhook URL:
      ```json
      {
        "name":"Suresh",
        "email":"johndeo@gmail.com",
        "city":"Delhi",
        "phone":"9836462538",
        "type":"3BHK Apartment",
        "budget":"4300000"
      }
      ```

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Sample Airtable resource | https://airtable.com/appe77qnWMLUWfKEe/shrQ9t6tboCuNkswq |
| Webhook payload example (name/email/city/phone/type/budget) | Included in “Sample Data & Resources” sticky note |
| Built-in setup reminders | “Workflow Overview” sticky note: connect Airtable, authenticate Gmail OAuth2, connect Slack OAuth2 |

