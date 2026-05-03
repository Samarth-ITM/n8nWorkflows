Create HubSpot support tickets from Tally form submissions

https://n8nworkflows.xyz/workflows/create-hubspot-support-tickets-from-tally-form-submissions-14474


# Create HubSpot support tickets from Tally form submissions

# 1. Workflow Overview

This workflow creates a HubSpot support ticket from a Tally form submission. Its primary goal is to ensure that every ticket is associated with a valid HubSpot contact: it first looks up the submitter by email, creates or updates the contact if necessary, then creates a support ticket linked to that contact.

Typical use cases include:

- Turning inbound support requests from a Tally form into HubSpot tickets
- Ensuring HubSpot contact records exist before ticket creation
- Standardizing customer intake from no-code forms into a CRM/helpdesk process

## 1.1 Input Reception and Contact Lookup

The workflow starts when a Tally form is submitted. It extracts the submitter’s email from the form payload and searches HubSpot contacts for a match.

## 1.2 Contact Existence Check and Contact Resolution

The workflow checks whether HubSpot returned a contact. If a contact exists, it proceeds with that record. If not, it creates or updates a contact in HubSpot using the submitted email.

## 1.3 Field Normalization and Ticket Creation

Because the workflow can arrive from two different branches, it normalizes the contact ID and name fields into a consistent structure, then creates a HubSpot support ticket associated with the resolved contact.

---

# 2. Block-by-Block Analysis

## Block 1 — Input Reception and Contact Lookup

### Overview

This block receives the Tally form submission and immediately performs a HubSpot contact lookup based on the submitted email address. It establishes whether the user already exists in HubSpot before any ticket is created.

### Nodes Involved

- When Tally Form Submitted
- Search HubSpot Contacts

### Node Details

#### 1. When Tally Form Submitted

- **Type and technical role:** `n8n-nodes-tallyforms.tallyTrigger`  
  Trigger node that starts the workflow whenever the configured Tally form receives a submission.
- **Configuration choices:**
  - Configured with a specific Tally form ID: `EkL9eB`
  - Uses Tally API credentials
- **Key expressions or variables used:**
  - This node provides the form payload used downstream
  - A key field later referenced is: `question_R4DYq4.value`, apparently representing the submitter email
- **Input and output connections:**
  - No input; this is an entry point
  - Outputs to: `Search HubSpot Contacts`
- **Version-specific requirements:**
  - Uses `typeVersion: 2`
  - Requires the Tally Trigger node available in the n8n instance
- **Edge cases or potential failure types:**
  - Invalid or disconnected Tally credentials
  - Wrong form ID causing no trigger events
  - Form schema changes in Tally causing downstream field references like `question_R4DYq4.value` to break
  - Webhook registration issues if the workflow is not properly activated
- **Sub-workflow reference:** None

#### 2. Search HubSpot Contacts

- **Type and technical role:** `n8n-nodes-base.hubspot`  
  HubSpot search node used to find an existing contact by email.
- **Configuration choices:**
  - Operation: `search`
  - Authentication: HubSpot app token
  - Filter group searches where property `email` matches the submitted Tally email
- **Key expressions or variables used:**
  - Filter value: `={{ $json.question_R4DYq4.value }}`
  - Property name is configured as `email|string`, which indicates a string-based email property search
- **Input and output connections:**
  - Input from: `When Tally Form Submitted`
  - Output to: `If Contact Already Exists`
- **Version-specific requirements:**
  - Uses `typeVersion: 2.2`
  - Requires HubSpot credentials compatible with the node’s app-token authentication mode
- **Edge cases or potential failure types:**
  - Invalid HubSpot app token
  - Missing or empty email field from Tally
  - Search returning no result
  - Search returning multiple results; workflow appears designed around a single item path, so duplicates in HubSpot may create ambiguity depending on HubSpot node behavior
  - Rate limiting or API errors from HubSpot
- **Sub-workflow reference:** None

---

## Block 2 — Contact Existence Check and Contact Resolution

### Overview

This block determines whether the Tally submitter already exists as a HubSpot contact. If a contact is found, the workflow uses that record directly; otherwise, it creates or updates a contact so that downstream ticket creation always has a contact ID.

### Nodes Involved

- If Contact Already Exists
- Upsert HubSpot Contact

### Node Details

#### 3. If Contact Already Exists

- **Type and technical role:** `n8n-nodes-base.if`  
  Conditional router that checks whether the HubSpot search result contains a contact ID.
- **Configuration choices:**
  - Condition type uses strict validation
  - Main test: `{{ $json.id }}` exists
  - If true: contact exists
  - If false: no contact found
- **Key expressions or variables used:**
  - `={{ $json.id }}`
- **Input and output connections:**
  - Input from: `Search HubSpot Contacts`
  - True output to: `Set Contact ID and Name`
  - False output to: `Upsert HubSpot Contact`
- **Version-specific requirements:**
  - Uses `typeVersion: 2.3`
  - Condition settings indicate version 3-style condition configuration
- **Edge cases or potential failure types:**
  - If the search node returns no items at all rather than an item without `id`, branch behavior may depend on n8n execution semantics
  - If HubSpot response shape changes or is incomplete, the condition may not behave as expected
  - Duplicate contacts could still pass this check without resolving which one is most appropriate
- **Sub-workflow reference:** None

#### 4. Upsert HubSpot Contact

- **Type and technical role:** `n8n-nodes-base.hubspot`  
  HubSpot contact create-or-update operation used when no existing contact is found.
- **Configuration choices:**
  - Configured as a contact upsert operation
  - Authentication: HubSpot app token
  - Email is taken from the original Tally trigger node, not from the immediate input item
  - No additional contact properties are currently mapped
- **Key expressions or variables used:**
  - Email: `={{ $('When Tally Form Submitted').item.json.question_R4DYq4.value }}`
- **Input and output connections:**
  - Input from: `If Contact Already Exists` false branch
  - Output to: `Set Contact ID and Name`
- **Version-specific requirements:**
  - Uses `typeVersion: 2.2`
  - Requires HubSpot app-token credentials with contact write permissions
- **Edge cases or potential failure types:**
  - Missing or malformed email
  - Permission issues when creating/updating contacts
  - If first name and last name are expected but not configured, contact records may be created with only email
  - Reliance on the trigger node reference means the expression assumes there is a corresponding item from the original Tally execution context
- **Sub-workflow reference:** None

---

## Block 3 — Field Normalization and Ticket Creation

### Overview

This block standardizes the contact data regardless of whether the workflow used an existing contact or created a new one. Once the fields are normalized into a consistent structure, it creates the HubSpot support ticket and associates it with the contact.

### Nodes Involved

- Set Contact ID and Name
- Create HubSpot Ticket

### Node Details

#### 5. Set Contact ID and Name

- **Type and technical role:** `n8n-nodes-base.set`  
  Data-shaping node that creates a normalized payload for downstream ticket creation.
- **Configuration choices:**
  - Assigns:
    - `id`
    - `properties.firstname`
    - `properties.lastname`
  - Pulls values from the current incoming item
- **Key expressions or variables used:**
  - `={{ $json.id }}`
  - `={{ $json.properties.firstname }}`
  - `={{ $json.properties.lastname }}`
- **Input and output connections:**
  - Input from:
    - `If Contact Already Exists` true branch
    - `Upsert HubSpot Contact`
  - Output to: `Create HubSpot Ticket`
- **Version-specific requirements:**
  - Uses `typeVersion: 3.4`
- **Edge cases or potential failure types:**
  - If the upstream node does not provide `properties.firstname` or `properties.lastname`, the ticket name may contain blank values
  - Depending on node settings, fields not explicitly assigned may be dropped, which is acceptable here but important if later extensions need original Tally data
  - If the HubSpot upsert response shape differs from the search response, missing nested `properties` may cause empty values
- **Sub-workflow reference:** None

#### 6. Create HubSpot Ticket

- **Type and technical role:** `n8n-nodes-base.hubspot`  
  HubSpot ticket creation node that creates a support ticket associated with the resolved contact.
- **Configuration choices:**
  - Resource: `ticket`
  - Pipeline ID: `0`
  - Stage ID: `1`
  - Ticket name format: `Support - {{ $json.properties.firstname }} {{ $json.properties.lastname }}`
  - Additional field `associatedContactIds` populated with the resolved contact ID as an array
- **Key expressions or variables used:**
  - Ticket name: `=Support - {{ $json.properties.firstname }} {{ $json.properties.lastname }}`
  - Associated contacts: `={{ [$json.id] }}`
- **Input and output connections:**
  - Input from: `Set Contact ID and Name`
  - No downstream node
- **Version-specific requirements:**
  - Uses `typeVersion: 2.2`
  - Requires HubSpot permissions for ticket creation and contact association
- **Edge cases or potential failure types:**
  - Invalid pipeline or stage ID for the target HubSpot portal
  - Missing contact ID causing association failure
  - Blank first/last names causing unattractive ticket names
  - HubSpot API permission errors or rate limiting
  - If associated contact IDs must exist in a specific format, malformed data may fail association
- **Sub-workflow reference:** None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note | `n8n-nodes-base.stickyNote` | Documentation/comment node |  |  | ## TEMPLATE - Tally form to Hubspot<br>### How it works<br>1. A Tally form submission triggers the workflow and searches HubSpot for an existing contact by email.<br>2. An If condition checks whether the contact was found; if not, a new contact is created or updated in HubSpot.<br>3. Fields are normalized (contact ID, first name, last name) regardless of which branch was taken.<br>4. A new HubSpot support ticket is created using the resolved contact data.<br>### Setup steps<br>- [ ] Connect your Tally account and configure the form trigger with the correct form ID.<br>- [ ] Connect your HubSpot account (API key or OAuth) for the Search contacts, Create or update a contact, and Create a ticket nodes.<br>- [ ] Map the Tally form fields (email, first name, last name) to the corresponding HubSpot contact properties in the 'Create or update a contact' node.<br>- [ ] Configure the 'Edit Fields' node to correctly map the contact ID and name fields from both the 'If' (existing contact) and 'Create or update a contact' (new contact) branches.<br>- [ ] Set the ticket properties (subject, status, pipeline, etc.) in the 'Create a ticket' node to match your HubSpot setup.<br>### Customization<br>You can extend the 'Edit Fields' node to pass additional Tally form fields (e.g., message, phone number) into the HubSpot ticket. You can also add an association step after ticket creation to link the ticket to the resolved contact. |
| Sticky Note1 | `n8n-nodes-base.stickyNote` | Documentation/comment node |  |  | ## Tally trigger and contact lookup<br>Receives the Tally form submission and immediately searches HubSpot to check whether the submitter already exists as a contact. |
| Sticky Note2 | `n8n-nodes-base.stickyNote` | Documentation/comment node |  |  | ## Contact existence check and upsert<br>Evaluates whether a matching HubSpot contact was found; if not, creates or updates the contact so a valid contact record always exists before proceeding. |
| Sticky Note3 | `n8n-nodes-base.stickyNote` | Documentation/comment node |  |  | ## Prepare fields and create ticket<br>Normalizes the contact ID and name fields from both branches, then creates a new HubSpot support ticket using the resolved contact data. |
| When Tally Form Submitted | `n8n-nodes-tallyforms.tallyTrigger` | Entry-point trigger on Tally form submission |  | Search HubSpot Contacts | ## Tally trigger and contact lookup<br>Receives the Tally form submission and immediately searches HubSpot to check whether the submitter already exists as a contact. |
| Search HubSpot Contacts | `n8n-nodes-base.hubspot` | Searches HubSpot contacts by submitted email | When Tally Form Submitted | If Contact Already Exists | ## Tally trigger and contact lookup<br>Receives the Tally form submission and immediately searches HubSpot to check whether the submitter already exists as a contact. |
| If Contact Already Exists | `n8n-nodes-base.if` | Routes execution based on whether a contact ID exists | Search HubSpot Contacts | Set Contact ID and Name; Upsert HubSpot Contact | ## Contact existence check and upsert<br>Evaluates whether a matching HubSpot contact was found; if not, creates or updates the contact so a valid contact record always exists before proceeding. |
| Upsert HubSpot Contact | `n8n-nodes-base.hubspot` | Creates or updates a HubSpot contact when none was found | If Contact Already Exists | Set Contact ID and Name | ## Contact existence check and upsert<br>Evaluates whether a matching HubSpot contact was found; if not, creates or updates the contact so a valid contact record always exists before proceeding. |
| Set Contact ID and Name | `n8n-nodes-base.set` | Normalizes contact fields for ticket creation | If Contact Already Exists; Upsert HubSpot Contact | Create HubSpot Ticket | ## Prepare fields and create ticket<br>Normalizes the contact ID and name fields from both branches, then creates a new HubSpot support ticket using the resolved contact data. |
| Create HubSpot Ticket | `n8n-nodes-base.hubspot` | Creates a HubSpot support ticket associated with the contact | Set Contact ID and Name |  | ## Prepare fields and create ticket<br>Normalizes the contact ID and name fields from both branches, then creates a new HubSpot support ticket using the resolved contact data. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - In n8n, create a blank workflow.
   - Optionally name it something like: `Tally form to HubSpot support ticket`.

2. **Add the Tally trigger node**
   - Add node: **Tally Trigger**
   - Rename it to: `When Tally Form Submitted`
   - Select or create Tally credentials
   - Set the form ID to your target Tally form, in this example: `EkL9eB`
   - Save the node
   - Important: verify the exact field names returned by your form submission payload. In this workflow, the email field is referenced as `question_R4DYq4.value`

3. **Add the HubSpot search node**
   - Add node: **HubSpot**
   - Rename it to: `Search HubSpot Contacts`
   - Connect `When Tally Form Submitted` → `Search HubSpot Contacts`
   - Configure:
     - **Operation:** Search
     - **Authentication:** App Token
   - In the search filters:
     - Add a filter for the contact email property
     - Set the value to the Tally email expression:
       `{{ $json.question_R4DYq4.value }}`
   - Select or create HubSpot app token credentials

4. **Add the If node**
   - Add node: **If**
   - Rename it to: `If Contact Already Exists`
   - Connect `Search HubSpot Contacts` → `If Contact Already Exists`
   - Configure the condition:
     - Check whether `{{ $json.id }}` **exists**
   - This creates:
     - **True branch** = contact found
     - **False branch** = no contact found

5. **Add the HubSpot contact upsert node**
   - Add node: **HubSpot**
   - Rename it to: `Upsert HubSpot Contact`
   - Connect the **false** output of `If Contact Already Exists` → `Upsert HubSpot Contact`
   - Configure it for contact create/update behavior
   - Set authentication to HubSpot app token
   - Set the email field to:
     `{{ $('When Tally Form Submitted').item.json.question_R4DYq4.value }}`
   - If desired, also map additional contact properties such as:
     - first name
     - last name
     - phone
   - Use values from your Tally form payload
   - Ensure the connected HubSpot credential has contact write access

6. **Add the Set/Edit Fields node**
   - Add node: **Set**
   - Rename it to: `Set Contact ID and Name`
   - Connect:
     - **True** output of `If Contact Already Exists` → `Set Contact ID and Name`
     - `Upsert HubSpot Contact` → `Set Contact ID and Name`
   - Add assignments:
     - `id` = `{{ $json.id }}`
     - `properties.firstname` = `{{ $json.properties.firstname }}`
     - `properties.lastname` = `{{ $json.properties.lastname }}`
   - This node standardizes the payload so the ticket node can use the same fields regardless of which branch executed

7. **Add the HubSpot ticket creation node**
   - Add node: **HubSpot**
   - Rename it to: `Create HubSpot Ticket`
   - Connect `Set Contact ID and Name` → `Create HubSpot Ticket`
   - Configure:
     - **Resource:** Ticket
     - **Pipeline ID:** `0`
     - **Stage ID:** `1`
     - **Ticket name:** `Support - {{ $json.properties.firstname }} {{ $json.properties.lastname }}`
   - In additional fields:
     - Set associated contact IDs to:
       `{{ [$json.id] }}`
   - Make sure the pipeline and stage IDs match your HubSpot environment

8. **Configure credentials**
   - **Tally credentials**
     - Connect a Tally account that has access to the target form
   - **HubSpot credentials**
     - Use a HubSpot app token credential compatible with the HubSpot node
     - Confirm scopes/permissions for:
       - Contact read/search
       - Contact create/update
       - Ticket create
       - Contact-ticket association

9. **Test the workflow**
   - Submit a Tally form response with a known email already present in HubSpot
   - Confirm the workflow:
     - finds the contact
     - skips contact creation
     - creates a ticket associated with the contact
   - Submit another response with a new email
   - Confirm the workflow:
     - does not find a contact
     - creates/updates the contact
     - creates the ticket

10. **Validate payload field names**
    - Inspect the Tally trigger output carefully
    - Replace `question_R4DYq4.value` with your real email field path if your form uses a different question ID
    - If you want first/last name in HubSpot and the ticket title, map those fields explicitly in `Upsert HubSpot Contact`

11. **Optional improvements**
    - Add validation before HubSpot search to reject empty email values
    - Add a fallback ticket name if first/last name are missing
    - Add error handling branches for HubSpot API failures
    - Add extra ticket properties such as description, priority, category, or source
    - Add an explicit association node if you need more advanced ticket-contact linking behavior

## Sub-workflow setup

This workflow does **not** use any Execute Workflow or sub-workflow nodes. There are no nested workflows to configure.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| The workflow uses Tally as the intake source and HubSpot as both the contact store and ticketing destination. | General architecture |
| The Tally field path used for email is `question_R4DYq4.value`; this is form-specific and will likely differ in another Tally form. | Implementation note |
| The sticky-note instructions mention mapping first name and last name in the contact upsert node, but the current node configuration only maps email. If you want reliable ticket names, add these mappings manually. | Configuration gap |
| The sticky-note instructions refer to an “Edit Fields” node and a “Create a ticket” node; in the actual workflow these are named `Set Contact ID and Name` and `Create HubSpot Ticket`. | Naming clarification |
| The workflow is currently inactive (`active: false` in the export), so it must be activated in n8n for the Tally trigger webhook to operate continuously. | Deployment note |