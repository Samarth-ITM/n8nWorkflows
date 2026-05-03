Sync Beex WhatsApp ticket messages with HubSpot contact activities

https://n8nworkflows.xyz/workflows/sync-beex-whatsapp-ticket-messages-with-hubspot-contact-activities-12744


# Sync Beex WhatsApp ticket messages with HubSpot contact activities

## 1. Workflow Overview

**Purpose:**  
This workflow listens to Beex “ticket management creation” events (specifically WhatsApp channel tickets), retrieves the full message history for the related ticket, formats each message into HTML (text/image/audio), consolidates them into a single chat transcript, and logs it in HubSpot as a **Communication** activity (`WHATS_APP`) associated to the matching HubSpot **Contact** (found by phone number).

**Target use cases:**
- Sync Beex WhatsApp conversation history into HubSpot timelines.
- Centralize customer conversation context inside HubSpot contact records.

### 1.1 Event Reception & Channel Filtering (Beex → n8n)
Receives Beex webhook events and only continues if the ticket channel is WhatsApp.

### 1.2 Contact Resolution (Phone → HubSpot Contact Search)
Builds a searchable phone string (manual country code + phone number) and searches HubSpot for the contact.

### 1.3 Ticket Message Retrieval (Beex Ticket → Messages)
Pulls all messages for the ticket referenced in the trigger event.

### 1.4 Message Routing + HTML Formatting (Text/Image/Audio)
Splits messages by type and formats each as HTML snippet including time/user/channel.

### 1.5 Consolidation + HubSpot Activity Logging
Orders messages chronologically, concatenates them, and creates a HubSpot Communication object associated to the contact.

---

## 2. Block-by-Block Analysis

### Block 1 — Event Reception & WhatsApp Filtering
**Overview:** Receives Beex events and enforces that only WhatsApp tickets are processed.  
**Nodes involved:** `Beex Trigger`, `Is WhatsApp Channel?`

#### Node: Beex Trigger
- **Type / role:** `n8n-nodes-beex.beexTrigger` (community node) — webhook trigger from Beex.
- **Configuration (interpreted):**
  - Listens for Beex event type: `management_create`.
  - Exposes a webhook endpoint (n8n-generated) to be registered in Beex Callback Integration.
- **Key data used later:** `data.ticket.id`, `data.ticket.channel.name`, `data.phone_number`.
- **Connections:**
  - Output → `Is WhatsApp Channel?`
- **Version-specific:** Type version `1`. Requires installing community package `n8n-nodes-beex`.
- **Potential failures / edge cases:**
  - Webhook not registered in Beex or wrong environment URL (test vs production).
  - Beex event payload shape differs (missing `data.ticket.*`).
  - Duplicate events (Beex retries) causing repeated HubSpot activities unless deduplication is added.

#### Node: Is WhatsApp Channel?
- **Type / role:** `Filter` — gate workflow by channel name.
- **Configuration (interpreted):**
  - Condition: `{{$json.data.ticket.channel.name}} == "WhatsApp Message"`
- **Connections:**
  - If matched → `Get Phone`
  - If not matched → workflow ends (no output path configured).
- **Version-specific:** Type version `2.2`.
- **Potential failures / edge cases:**
  - Channel naming mismatch (e.g., “WhatsApp” vs “WhatsApp Message”) will block valid events.
  - Missing `data.ticket.channel.name` will evaluate to empty/undefined and fail the equals check.

---

### Block 2 — Contact Resolution (Beex phone → HubSpot contact)
**Overview:** Builds a normalized phone string and searches HubSpot for a contact using that phone value.  
**Nodes involved:** `Get Phone`, `Search Contact`

#### Node: Get Phone
- **Type / role:** `Set` — creates/overrides fields used for HubSpot search.
- **Configuration (interpreted):**
  - Sets `country_code` to a hardcoded placeholder `*insert country_code*` (must be edited).
  - Sets `phone` from trigger payload: `{{$json.data.phone_number}}`
- **Connections:**
  - Output → `Search Contact`
- **Version-specific:** Type version `3.4`.
- **Potential failures / edge cases:**
  - If `country_code` is not replaced with a real prefix (e.g., `+33`), HubSpot search may not match.
  - Phone formats can differ between Beex and HubSpot (`+`, spaces, leading zeros). Consider normalizing (remove spaces, ensure E.164).

#### Node: Search Contact
- **Type / role:** `HubSpot` node — searches CRM contacts.
- **Configuration (interpreted):**
  - Authentication: **HubSpot App Token** (private app token).
  - Operation: `search`, limit `1`.
  - Filter: property `phone` equals `{{$json.country_code}}{{$json.phone}}`
  - Requests additional properties: `firstname`, `phone` (for output context).
  - Retries enabled (`maxTries: 2`, `waitBetweenTries: 2500ms`).
- **Connections:**
  - Output → `Get Messages`
- **Version-specific:** Type version `2.2`.
- **Potential failures / edge cases:**
  - No matching contact: workflow stops here because `alwaysOutputData` is `false` (no downstream execution).
  - HubSpot token missing permissions (needs at least contact read; later step needs communication write).
  - Phone property might not be indexed/consistent; equality search is strict—consider “contains” or normalization.
  - Rate limiting (HubSpot 429) — retry may still fail on bursts.

---

### Block 3 — Retrieve Ticket Messages from Beex
**Overview:** Fetches all messages for the ticket ID that triggered the event.  
**Nodes involved:** `Get Messages`

#### Node: Get Messages
- **Type / role:** `n8n-nodes-beex.beex` (community node) — calls Beex API to fetch ticket messages.
- **Configuration (interpreted):**
  - Resource: `tickets`
  - Operation: `get-messages`
  - Ticket ID: `={{ $('Beex Trigger').item.json.data.ticket.id }}`
  - Uses Beex API token credentials.
  - Retries enabled (`waitBetweenTries: 2500ms`).
- **Connections:**
  - Output → `Routes`
- **Version-specific:** Type version `1`. Requires Beex community node + valid API key.
- **Potential failures / edge cases:**
  - Missing/invalid `ticket.id` in trigger payload.
  - Beex auth failure (expired token / wrong API key).
  - Large message history could increase execution time and payload size.

---

### Block 4 — Route by Message Type & Format to HTML
**Overview:** Classifies each message item as text, image, or audio based on fields present and media URL extension; produces standardized HTML fragments for later consolidation.  
**Nodes involved:** `Routes`, `Format Text`, `Format Image`, `Format Audio`, `Merge`

#### Node: Routes
- **Type / role:** `Switch` — routes each message item into one of three outputs.
- **Configuration (interpreted):**
  - **Text** route: `{{$json.message}}` is not empty.
  - **Image** route: `{{$json.media_url}}` matches image extensions regex: `\.(png|jpe?g|gif|bmp|webp|svg|tiff|avif|heic|ico)$`
  - **Audio** route: `{{$json.media_url}}` matches audio extensions regex: `\.(mp3|wav|ogg|flac|aac|m4a|wma|opus|aiff?|alac|midi?)$`
  - Outputs are renamed: `Text`, `Image`, `Audio`.
- **Connections:**
  - Text → `Format Text`
  - Image → `Format Image`
  - Audio → `Format Audio`
- **Version-specific:** Type version `3.3`.
- **Potential failures / edge cases:**
  - Media URLs with query strings (`.jpg?token=...`) won’t match the regex (because `$` anchors end-of-string).
  - A message might have both `message` and `media_url`; only the first matching rule effectively routes if multiple could match (depending on Switch behavior/settings).
  - Other media types (video, pdf) are not handled and will be dropped.

#### Node: Format Text
- **Type / role:** `Set` — builds an HTML snippet for text messages.
- **Configuration (interpreted):**
  - Builds `message` HTML using:
    - `time = $json.created_at.slice(11,16)` (expects ISO-like string)
    - `user = $json.person?.full_name ?? "Unknown"`
    - `channel = $json.channel?.name ?? "Unknown"`
    - `message = $json.message.replace(/[\r\n]+/g," ")`
    - Output format: `<b>{time} {user} ({channel}):</b><br/>{message}<br/><br/>`
  - Also sets `created_at` for sorting: `{{$json.created_at}}`
- **Connections:** Output → `Merge` (input 1 / index 0)
- **Version-specific:** Type version `3.4`.
- **Potential failures / edge cases:**
  - If `created_at` is missing/short, `.slice(11,16)` can produce incorrect strings.
  - If `$json.message` is null, `.replace(...)` can throw (though routing checks “not empty”, it still assumes string).

#### Node: Format Image
- **Type / role:** `Set` — builds HTML snippet embedding the image.
- **Configuration (interpreted):**
  - Uses similar header (time/user/channel).
  - Embeds: `<img src={media_url} width=350><br/><br/>`
  - Sets `created_at`.
- **Connections:** Output → `Merge` (input 2 / index 1)
- **Version-specific:** Type version `3.4`.
- **Potential failures / edge cases:**
  - HTML attribute quoting: `src=` is not wrapped in quotes; URLs containing `&` or spaces may break rendering. Safer: `<img src="${media_url}" ...>`.
  - If `media_url` is empty or not publicly accessible, HubSpot may show broken image.

#### Node: Format Audio
- **Type / role:** `Set` — builds HTML snippet referencing audio by filename (not embedding player).
- **Configuration (interpreted):**
  - Header uses channel name + `" Audio"`.
  - Message text: `"Audio-" + $json.media_url.split("/").last()`
  - Sets `created_at`.
- **Connections:** Output → `Merge` (input 3 / index 2)
- **Version-specific:** Type version `3.4`.
- **Potential failures / edge cases:**
  - `.last()` is not standard JavaScript; n8n expressions provide some helpers but can vary by version. If unsupported, this will fail. Safer expression: `{{$json.media_url.split('/').pop()}}`.
  - Audio URL isn’t linked; only filename is logged (may be insufficient for users).

#### Node: Merge
- **Type / role:** `Merge` — appends the three formatted streams into one combined stream.
- **Configuration (interpreted):**
  - Mode: “append” behavior via `numberInputs: 3` and downstream sort.
- **Connections:** Output → `Sort Messages`
- **Version-specific:** Type version `3.2`.
- **Potential failures / edge cases:**
  - If one branch produces no items, merge still works, but ordering depends on later sort.
  - If “append” semantics change, you may need explicit merge mode settings.

---

### Block 5 — Sort, Consolidate, and Log to HubSpot
**Overview:** Orders all formatted snippets by timestamp, concatenates them into one HTML body, then creates a HubSpot Communication object associated to the found contact.  
**Nodes involved:** `Sort Messages`, `Consolidate Chat`, `Register Activity`

#### Node: Sort Messages
- **Type / role:** `Sort` — chronological ordering.
- **Configuration (interpreted):**
  - Sort field: `created_at` (ascending default).
- **Connections:** Output → `Consolidate Chat`
- **Version-specific:** Type version `1`.
- **Potential failures / edge cases:**
  - String sorting may not match chronological order if `created_at` isn’t ISO-8601 sortable. (If it is ISO, lexical sort works.)
  - Missing `created_at` on any item may affect ordering.

#### Node: Consolidate Chat
- **Type / role:** `Code` — builds one combined “chat transcript” string.
- **Configuration (interpreted):**
  - Iterates over all incoming items.
  - Concatenates `item.json.message` into `chat`.
  - Removes all double quotes via `replaceAll('"','')`.
  - Returns a single item: `{ chat }`.
- **Connections:** Output → `Register Activity`
- **Version-specific:** Type version `2`.
- **Potential failures / edge cases:**
  - If the incoming HTML intentionally contains quotes (e.g., image `src="..."`), removing quotes can break HTML. (Currently image formatter doesn’t add quotes, but this is fragile.)
  - Large histories may exceed HubSpot property limits for `hs_communication_body`.

#### Node: Register Activity
- **Type / role:** `HTTP Request` — direct call to HubSpot CRM v3 Communications API.
- **Configuration (interpreted):**
  - Method: `POST`
  - URL: `https://api.hubapi.com/crm/v3/objects/communications`
  - Authentication: predefined credential type `hubspotAppToken`
  - JSON body creates a Communication with:
    - `hs_communication_channel_type`: `WHATS_APP`
    - `hs_communication_logged_from`: `CRM`
    - `hs_communication_body`: `{{$json.chat}}`
    - `hs_timestamp`: `{{$now.toMillis()}}`
  - Association:
    - Associates to Contact id: `{{ $('Search Contact').first().json.id }}`
    - Uses `associationTypeId: 81` (HubSpot-defined association for Communication → Contact).
  - Retries enabled (`maxTries: 2`, `waitBetweenTries: 2500ms`).
- **Connections:** Terminal node (no outputs).
- **Version-specific:** Type version `4.3`.
- **Potential failures / edge cases:**
  - If `Search Contact` returned no items, `first().json.id` expression fails and the request errors.
  - HubSpot permissions: private app must allow CRM object write for communications + association rights.
  - HubSpot API may reject oversized `hs_communication_body` or invalid HTML.
  - Rate limits / transient 5xx.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Beex Trigger | n8n-nodes-beex.beexTrigger | Receives Beex webhook event (`management_create`) | — | Is WhatsApp Channel? | ## Beex Trigger Node + Filter - The trigger receives the `On Management Create` event - Currently, the workflow only accepts an event that uses WhatsApp messaging as its **channel** |
| Is WhatsApp Channel? | n8n-nodes-base.filter | Filter only WhatsApp ticket channel events | Beex Trigger | Get Phone | ## Beex Trigger Node + Filter - The trigger receives the `On Management Create` event - Currently, the workflow only accepts an event that uses WhatsApp messaging as its **channel** |
| Get Phone | n8n-nodes-base.set | Build phone search key (country code + phone) | Is WhatsApp Channel? | Search Contact | ## Retrieving Messages from BeexCC - We strategically obtain a valid phone number (*insert* the **country code**) - We are looking for HubSpot's contact information using their **phone number** - We obtain the messages created through the given interaction in BeexCC (use the ticket id of the initial trigger node) |
| Search Contact | n8n-nodes-base.hubspot | Find HubSpot contact by phone | Get Phone | Get Messages | ## Retrieving Messages from BeexCC - We strategically obtain a valid phone number (*insert* the **country code**) - We are looking for HubSpot's contact information using their **phone number** - We obtain the messages created through the given interaction in BeexCC (use the ticket id of the initial trigger node) |
| Get Messages | n8n-nodes-beex.beex | Fetch Beex ticket message history | Search Contact | Routes | ## Retrieving Messages from BeexCC - We strategically obtain a valid phone number (*insert* the **country code**) - We are looking for HubSpot's contact information using their **phone number** - We obtain the messages created through the given interaction in BeexCC (use the ticket id of the initial trigger node) |
| Routes | n8n-nodes-base.switch | Route each message by type (text/image/audio) | Get Messages | Format Text; Format Image; Format Audio | ## Routing by message type - Depending on the type of message (text, image or audio) sent during the interaction, it will be given an equivalent HTML format. Then we consolidated. |
| Format Text | n8n-nodes-base.set | Convert text message to HTML snippet + created_at | Routes (Text) | Merge | ## Routing by message type - Depending on the type of message (text, image or audio) sent during the interaction, it will be given an equivalent HTML format. Then we consolidated. |
| Format Image | n8n-nodes-base.set | Convert image message to HTML snippet + created_at | Routes (Image) | Merge | ## Routing by message type - Depending on the type of message (text, image or audio) sent during the interaction, it will be given an equivalent HTML format. Then we consolidated. |
| Format Audio | n8n-nodes-base.set | Convert audio message to HTML snippet + created_at | Routes (Audio) | Merge | ## Routing by message type - Depending on the type of message (text, image or audio) sent during the interaction, it will be given an equivalent HTML format. Then we consolidated. |
| Merge | n8n-nodes-base.merge | Append all formatted messages into one stream | Format Text; Format Image; Format Audio | Sort Messages | ## Routing by message type - Depending on the type of message (text, image or audio) sent during the interaction, it will be given an equivalent HTML format. Then we consolidated. |
| Sort Messages | n8n-nodes-base.sort | Sort formatted messages by created_at | Merge | Consolidate Chat | ## Logging Activity in HubSpot - We proceed to sort the messages by creation. - We consolidate the messages into a single record. - We record the activity of type `WHATS_APP` in HubSpot. |
| Consolidate Chat | n8n-nodes-base.code | Concatenate HTML snippets into one `chat` field | Sort Messages | Register Activity | ## Logging Activity in HubSpot - We proceed to sort the messages by creation. - We consolidate the messages into a single record. - We record the activity of type `WHATS_APP` in HubSpot. |
| Register Activity | n8n-nodes-base.httpRequest | Create HubSpot Communication and associate to Contact | Consolidate Chat | — | ## Logging Activity in HubSpot - We proceed to sort the messages by creation. - We consolidate the messages into a single record. - We record the activity of type `WHATS_APP` in HubSpot. |
| Sticky Note | n8n-nodes-base.stickyNote | Comment block | — | — | ## Routing by message type - Depending on the type of message (text, image or audio) sent during the interaction, it will be given an equivalent HTML format. Then we consolidated. |
| Sticky Note1 | n8n-nodes-base.stickyNote | Comment block | — | — | ## Beex Trigger Node + Filter - The trigger receives the `On Management Create` event - Currently, the workflow only accepts an event that uses WhatsApp messaging as its **channel** |
| Sticky Note2 | n8n-nodes-base.stickyNote | Comment block | — | — | ## Logging Activity in HubSpot - We proceed to sort the messages by creation. - We consolidate the messages into a single record. - We record the activity of type `WHATS_APP` in HubSpot. |
| Sticky Note3 | n8n-nodes-base.stickyNote | Comment block | — | — | ## Retrieving Messages from BeexCC - We strategically obtain a valid phone number (*insert* the **country code**) - We are looking for HubSpot's contact information using their **phone number** - We obtain the messages created through the given interaction in BeexCC (use the ticket id of the initial trigger node) |
| Sticky Note4 | n8n-nodes-base.stickyNote | Comment block | — | — | # Sync Beex Ticket Messages to HubSpot Activities \n> **Requires:** Community node `n8n-nodes-beex`\n\n## Overview\nCaptures WhatsApp ticket classifications from Beex and syncs message history as HubSpot contact activities.\n\n## How it works \n1. **Beex Trigger** - Receives ticket classification events (On Management Create)\n2. **Filter** - Processes only WhatsApp Message channel events\n3. **Get Phone** - Extracts phone number (requires manual country code config)\n4. **Search Contact** - Locates HubSpot contact by phone number\n5. **Get Messages** - Retrieves all interaction messages using ticket ID\n6. **Route & Format** - Processes text/image/audio messages into HTML\n7. **Sort** - Orders messages by `created_at` timestamp\n8. **Consolidate** - Combines all messages into single HTML record for `hs_communication_body`\n9. **Register Activity** - Using the HubSpot API\n\n ## Setup steps\n### 1. Install Beex Package\n`n8n-nodes-beex`\n### 2. HubSpot Credentials\n- Create private app token with **Contacts** read/write permissions\n- Add credentials to HubSpot nodes\n### 3. Beex Configuration\n- Navigate to **Platform Settings → API Key and Callback**\n- Copy API key → paste into \"Get Messages\" node (`YOUR_TOKEN_HERE`)\n- Enable **Typing Registry in Callback Integration**\n### 4. Webhook Setup\n- Copy webhook URL from Beex trigger node (Test/Production)\n- Paste into Beex **Callback Integration** section\n- Save changes |

---

## 4. Reproducing the Workflow from Scratch

1. **Install the Beex community node**
   - In n8n, install community package: `n8n-nodes-beex`
   - Restart n8n if required.

2. **Create credentials**
   - **Beex API credential**
     - Create a Beex API token in Beex (Platform Settings → API Key and Callback).
     - In n8n: Credentials → create **Beex API** credential → paste token.
   - **HubSpot App Token credential**
     - In HubSpot: create a **Private App** and generate an access token.
     - Ensure permissions include at least:
       - Contacts: read (search) (and write if you plan to create/update contacts)
       - CRM objects / Communications: write (for logging)
     - In n8n: Credentials → create **HubSpot App Token** credential → paste token.

3. **Add node: “Beex Trigger”**
   - Node type: **Beex Trigger** (`n8n-nodes-beex.beexTrigger`)
   - Event types: select `management_create`
   - Copy the generated webhook URL (test and/or production).

4. **Configure Beex to call the webhook**
   - In Beex Callback Integration:
     - Paste the n8n webhook URL.
     - Enable required callback options (as per your Beex setup; note mentions “Typing Registry in Callback Integration”).
   - Save and send a test event to confirm n8n receives it.

5. **Add node: “Is WhatsApp Channel?” (Filter)**
   - Condition: String equals
     - Left: `{{$json.data.ticket.channel.name}}`
     - Right: `WhatsApp Message`
   - Connect: `Beex Trigger` → `Is WhatsApp Channel?`

6. **Add node: “Get Phone” (Set)**
   - Add fields:
     - `country_code` (string): set to your value, e.g. `+33` (replace placeholder)
     - `phone` (string): `{{$json.data.phone_number}}`
   - Connect: `Is WhatsApp Channel?` → `Get Phone`

7. **Add node: “Search Contact” (HubSpot)**
   - Node type: **HubSpot**
   - Authentication: **App Token**
   - Operation: **Search**
   - Limit: `1`
   - Filter:
     - Property: `phone`
     - Operator: equals
     - Value: `{{$json.country_code}}{{$json.phone}}`
   - Additional properties to return: `firstname`, `phone`
   - Connect: `Get Phone` → `Search Contact`

8. **Add node: “Get Messages” (Beex)**
   - Node type: **Beex** (`n8n-nodes-beex.beex`)
   - Resource: `tickets`
   - Operation: `get-messages`
   - Ticket ID expression: `{{ $('Beex Trigger').item.json.data.ticket.id }}`
   - Connect: `Search Contact` → `Get Messages`

9. **Add node: “Routes” (Switch)**
   - Create 3 rules (renamed outputs):
     - **Text**: `{{$json.message}}` is not empty
     - **Image**: `{{$json.media_url}}` matches regex for image extensions
     - **Audio**: `{{$json.media_url}}` matches regex for audio extensions
   - Connect: `Get Messages` → `Routes`

10. **Add formatting nodes (Set)**
    - **Format Text**: build HTML snippet + set `created_at`
    - **Format Image**: build HTML snippet with `<img ...>` + set `created_at`
    - **Format Audio**: build HTML snippet + set `created_at`
    - Connect:
      - `Routes: Text` → `Format Text`
      - `Routes: Image` → `Format Image`
      - `Routes: Audio` → `Format Audio`

11. **Add node: “Merge”**
    - Node type: **Merge**
    - Number of inputs: `3`
    - Connect:
      - `Format Text` → `Merge` (input 1)
      - `Format Image` → `Merge` (input 2)
      - `Format Audio` → `Merge` (input 3)

12. **Add node: “Sort Messages”**
    - Node type: **Sort**
    - Sort by field: `created_at`
    - Connect: `Merge` → `Sort Messages`

13. **Add node: “Consolidate Chat” (Code)**
    - Node type: **Code**
    - Code logic: iterate all items and concatenate `item.json.message` into `chat`, then return `{ chat }`.
    - Connect: `Sort Messages` → `Consolidate Chat`

14. **Add node: “Register Activity” (HTTP Request)**
    - Node type: **HTTP Request**
    - Method: `POST`
    - URL: `https://api.hubapi.com/crm/v3/objects/communications`
    - Authentication: **Predefined Credential Type** → `hubspotAppToken`
    - Body type: JSON
    - Body includes:
      - `hs_communication_channel_type = WHATS_APP`
      - `hs_communication_logged_from = CRM`
      - `hs_communication_body = {{$json.chat}}`
      - `hs_timestamp = {{$now.toMillis()}}`
      - Association to contact id: `{{ $('Search Contact').first().json.id }}`
      - Association type id: `81`
    - Connect: `Consolidate Chat` → `Register Activity`

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Requires community node `n8n-nodes-beex` | Mentioned in workflow notes (Sticky Note4) |
| Beex webhook must be configured in Beex Callback Integration | Copy the webhook URL from Beex Trigger node (test/prod) into Beex |
| Phone matching requires manual `country_code` configuration | `Get Phone` node contains `*insert country_code*` placeholder |
| HubSpot activity is created as a Communication with channel type `WHATS_APP` | `Register Activity` node posts to `/crm/v3/objects/communications` and associates to Contact |

Disclaimer: Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.