Streamline lead intake and AI routing using Typeform, OpenAI & Gmail

https://n8nworkflows.xyz/workflows/streamline-lead-intake-and-ai-routing-using-typeform--openai---gmail-14363


# Streamline lead intake and AI routing using Typeform, OpenAI & Gmail

## 1. Workflow Overview

This workflow captures inbound leads from a Typeform form, standardizes the submitted data, stores it in Airtable, uses OpenAI to classify the lead, then routes the lead into different follow-up email paths. For sales leads, it also sends a Discord alert to the team.

Its main use case is automated lead intake for service businesses or agencies that want to:
- collect inquiries from a Typeform form,
- keep a durable record in Airtable,
- enrich the lead with AI-based categorization and scoring,
- send tailored responses automatically,
- notify the internal team for high-value sales opportunities.

### 1.1 Input Reception and Data Cleaning
The workflow starts when a Typeform submission is received. A Code node then normalizes the name, email, and message fields and extracts the sender’s email domain.

### 1.2 Persistent Lead Storage
The cleaned lead is immediately saved to Airtable with a default status of `New`. This acts as the durable system of record even if downstream steps fail.

### 1.3 AI Classification and Data Combination
The saved Airtable record is analyzed by OpenAI using a strict prompt that asks for JSON output containing category, lead score, and priority. The AI output is then merged with the Airtable record.

### 1.4 Routing and Notifications
A Switch node routes the lead based on category. Each route sends a tailored Gmail response. The sales path additionally sends a Discord message to notify the internal team.

---

## 2. Block-by-Block Analysis

## 2.1 Block: Input Reception and Data Cleaning

**Overview:**  
This block receives a new Typeform submission and converts the raw form answers into normalized fields suitable for CRM storage and later processing. It also derives the email domain, which may be useful for enrichment or filtering later.

**Nodes Involved:**  
- Typeform Trigger
- Sanitize Lead Data

### Node: Typeform Trigger

- **Type and technical role:**  
  `n8n-nodes-base.typeformTrigger`  
  Entry-point trigger node that listens for new submissions on a specific Typeform form.

- **Configuration choices:**  
  - Configured with form ID `UBdasOKD`.
  - Uses a webhook-based trigger pattern.

- **Key expressions or variables used:**  
  - No custom expressions in parameters.
  - Downstream nodes expect raw Typeform answers under human-readable question labels.

- **Input and output connections:**  
  - Input: none; this is a trigger.
  - Output: connected to `Sanitize Lead Data`.

- **Version-specific requirements:**  
  - Type version `1.1`.
  - Requires valid Typeform credentials in n8n.
  - The form structure must match the field labels expected by the Code node.

- **Edge cases or potential failure types:**  
  - Trigger may fail if the Typeform credential expires or is revoked.
  - If the Typeform question text changes, downstream extraction may break because the Code node references exact question labels.
  - Webhook registration issues can occur if the workflow is inactive or the trigger was not properly registered.

- **Sub-workflow reference:**  
  - None.

---

### Node: Sanitize Lead Data

- **Type and technical role:**  
  `n8n-nodes-base.code`  
  JavaScript transformation node used to clean and normalize the incoming lead payload.

- **Configuration choices:**  
  - Reads these exact incoming fields:
    - `What is your full name?`
    - `What is your email address?`
    - `What project can we help you with today?`
  - Produces standardized output:
    - `Name`
    - `Email`
    - `Message`
    - `Domain`
  - Name is converted to title case.
  - Email is lowercased and trimmed.
  - Domain is extracted from the portion after `@`.

- **Key expressions or variables used:**  
  - JavaScript references via `$json[...]`.
  - Uses fallback empty strings for missing values.
  - Includes custom helper function `toTitleCase(str)`.

- **Input and output connections:**  
  - Input: `Typeform Trigger`
  - Output: `Create a record`

- **Version-specific requirements:**  
  - Type version `2`.
  - Requires JavaScript execution support in the Code node.

- **Edge cases or potential failure types:**  
  - If Typeform question labels are renamed, fields will become empty.
  - `toTitleCase` is simplistic and may not preserve names with apostrophes, hyphens, initials, acronyms, or locale-specific casing.
  - Invalid or missing email produces an empty or malformed `Domain`.
  - No explicit validation is applied before forwarding to Airtable.

- **Sub-workflow reference:**  
  - None.

---

## 2.2 Block: Persistent Lead Storage

**Overview:**  
This block writes the normalized lead into Airtable immediately after intake. It ensures the lead is preserved with a default status even if later AI, routing, or notification steps fail.

**Nodes Involved:**  
- Create a record

### Node: Create a record

- **Type and technical role:**  
  `n8n-nodes-base.airtable`  
  Creates a new Airtable row representing the lead.

- **Configuration choices:**  
  - Operation: `create`
  - Base: selected via credential-linked resource list
  - Table: selected via credential-linked resource list
  - Mapped columns:
    - `Name` ← `{{$json.Name}}`
    - `Email` ← `{{$json.Email}}`
    - `Status` ← static value `New`
    - `Message` ← `{{$json.Message}}`
  - Mapping mode is automatic input mapping.

- **Key expressions or variables used:**  
  - `={{ $json.Name }}`
  - `={{ $json.Email }}`
  - `={{ $json.Message }}`

- **Input and output connections:**  
  - Input: `Sanitize Lead Data`
  - Output:
    - `AI Lead Analyzer`
    - `Merge` input 1

- **Version-specific requirements:**  
  - Type version `2.1`.
  - Requires Airtable credentials and selection of a valid base and table.
  - The target Airtable must contain compatible fields at minimum:
    - Name
    - Email
    - Message
    - Status

- **Edge cases or potential failure types:**  
  - Missing credentials or revoked API access.
  - Base/table not selected or deleted.
  - Schema mismatch if expected columns do not exist.
  - Status field is configured as an Airtable options field with values `New` and `Done`; if the Airtable field values differ, creation may fail.
  - This node does **not** write the AI-derived fields `Category`, `Lead Score`, or `Priority`, even though later nodes seem to expect them under `fields`.

- **Sub-workflow reference:**  
  - None.

---

## 2.3 Block: AI Classification and Data Combination

**Overview:**  
This block asks OpenAI to classify the lead message and assign a score and priority, then combines the AI result with the Airtable record. It is the intelligence layer of the workflow.

**Nodes Involved:**  
- AI Lead Analyzer
- Merge

### Node: AI Lead Analyzer

- **Type and technical role:**  
  `@n8n/n8n-nodes-langchain.openAi`  
  OpenAI chat/completion-style node used to classify a lead message.

- **Configuration choices:**  
  - Model: `gpt-4.1-mini`
  - Prompt instructs the model to:
    - classify into one of:
      - Sales
      - Support
      - Partnership
      - Feedback
      - Other
    - generate `lead_score` from 1 to 10
    - assign `priority` as Low, Medium, or High
  - Prompt explicitly requires:
    - strict JSON only,
    - no markdown,
    - no explanatory text,
    - no extra fields.

- **Key expressions or variables used:**  
  - Prompt injects:
    - `{{$json["Message"]}}`

- **Input and output connections:**  
  - Input: `Create a record`
  - Output: `Merge` input 0

- **Version-specific requirements:**  
  - Type version `2.1`
  - Requires OpenAI credentials configured in n8n.
  - Availability of `gpt-4.1-mini` depends on account/model access.

- **Edge cases or potential failure types:**  
  - If the incoming item does not expose `Message` at the top level, the prompt receives an empty value.
  - LLM output may still deviate from valid JSON despite the prompt.
  - Network errors, rate limits, or quota exhaustion can break execution.
  - There is no dedicated parser node after the AI response, so downstream behavior depends heavily on the exact structure emitted by this node.
  - The sticky note says “GPT-4o-mini,” but the actual node uses `gpt-4.1-mini`.

- **Sub-workflow reference:**  
  - None.

---

### Node: Merge

- **Type and technical role:**  
  `n8n-nodes-base.merge`  
  Combines the AI output with the Airtable record into a single item.

- **Configuration choices:**  
  - Mode: `combine`
  - Combine strategy: `combineByPosition`
  - This means the first item from input 0 is combined with the first item from input 1.

- **Key expressions or variables used:**  
  - No explicit expressions in this node.

- **Input and output connections:**  
  - Input 0: `AI Lead Analyzer`
  - Input 1: `Create a record`
  - Output: `Route by Category`

- **Version-specific requirements:**  
  - Type version `3.2`

- **Edge cases or potential failure types:**  
  - If one branch returns no items, merge output may be empty or malformed for downstream logic.
  - If AI output structure is nested differently than expected, later references like `$json.fields.Category` will fail.
  - Combining by position assumes one item per lead on both branches; bulk or batched executions can misalign if counts diverge.

- **Sub-workflow reference:**  
  - None.

---

## 2.4 Block: Routing and Outbound Responses

**Overview:**  
This block decides which acknowledgement email to send based on category and, for the sales path, notifies the team on Discord. It is the operational response layer of the workflow.

**Nodes Involved:**  
- Route by Category
- Send a message
- Send a message1
- Send a message2
- Send a message3
- Send a message4

### Node: Route by Category

- **Type and technical role:**  
  `n8n-nodes-base.switch`  
  Branching node that routes one item into one of several category-specific outputs.

- **Configuration choices:**  
  - Uses four rules:
    1. `{{$json.fields.Category}} == "Sales"`
    2. `{{$json.fields.Category}} == "Support"`
    3. `{{$json.fields.Category}} == "Partnership"`
    4. `{{$json.fields.Category}} == "Default"`
  - Each rule maps to a separate output.

- **Key expressions or variables used:**  
  - `={{$json.fields.Category}}`

- **Input and output connections:**  
  - Input: `Merge`
  - Outputs:
    - Output 0 → `Send a message`
    - Output 1 → `Send a message1`
    - Output 2 → `Send a message2`
    - Output 3 → `Send a message3`

- **Version-specific requirements:**  
  - Type version `3.4`

- **Edge cases or potential failure types:**  
  - Important likely issue: the AI prompt outputs `category` in lowercase key form, but this Switch expects `fields.Category`.
  - Important likely issue: Airtable create output only contains submitted fields unless the table itself has those fields populated; the workflow does not update Airtable with AI results.
  - The fourth branch checks for `Default`, but the AI prompt never outputs `Default`; it outputs `Feedback` or `Other`.
  - Therefore, as currently configured, `Feedback` and `Other` leads likely do not match any branch.
  - If no rule matches, the item may stop here unless fallback handling is added.

- **Sub-workflow reference:**  
  - None.

---

### Node: Send a message

- **Type and technical role:**  
  `n8n-nodes-base.gmail`  
  Sends the sales-specific acknowledgment email.

- **Configuration choices:**  
  - Recipient: `{{$json.fields.Email}}`
  - Subject: `🔥 Let's Build Something Great, {{$json.fields.Name}}`
  - Body is plain-text style with a placeholder services link:
    - `[Your Services Link]`

- **Key expressions or variables used:**  
  - `={{ $json.fields.Email }}`
  - `{{$json.fields.Name}}`

- **Input and output connections:**  
  - Input: `Route by Category` sales output
  - Output: `Send a message4`

- **Version-specific requirements:**  
  - Type version `2.2`
  - Requires Gmail credentials with send permissions.

- **Edge cases or potential failure types:**  
  - If `fields.Email` or `fields.Name` is absent, email send may fail or produce malformed content.
  - Placeholder link is not a real URL and should be replaced before production.
  - Gmail quota or OAuth expiration can stop sending.

- **Sub-workflow reference:**  
  - None.

---

### Node: Send a message1

- **Type and technical role:**  
  `n8n-nodes-base.gmail`  
  Sends the support acknowledgment email.

- **Configuration choices:**  
  - Recipient: `{{$json.fields.Email}}`
  - Subject: `We Got Your Request 👍`
  - HTML-formatted body includes:
    - customer name,
    - quoted message,
    - note that the inquiry is logged as `NEW`.

- **Key expressions or variables used:**  
  - `={{ $json.fields.Email }}`
  - `{{ $json.fields.Name }}`
  - `{{ $json.fields.Message }}`

- **Input and output connections:**  
  - Input: `Route by Category` support output
  - Output: none

- **Version-specific requirements:**  
  - Type version `2.2`
  - Requires Gmail credentials.

- **Edge cases or potential failure types:**  
  - Same field-path issue as above if `fields.*` does not exist.
  - Unescaped HTML-sensitive user content may render unexpectedly, depending on Gmail node handling.
  - OAuth or send quota errors possible.

- **Sub-workflow reference:**  
  - None.

---

### Node: Send a message2

- **Type and technical role:**  
  `n8n-nodes-base.gmail`  
  Sends the partnership acknowledgment email.

- **Configuration choices:**  
  - Recipient: `{{$json.fields.Email}}`
  - Subject: `Partnership Opportunity 🚀`
  - Body is short, plain text, collaboration-focused.

- **Key expressions or variables used:**  
  - `={{ $json.fields.Email }}`
  - `{{$json.fields.Name}}`

- **Input and output connections:**  
  - Input: `Route by Category` partnership output
  - Output: none

- **Version-specific requirements:**  
  - Type version `2.2`
  - Requires Gmail credentials.

- **Edge cases or potential failure types:**  
  - Same missing-field risks on `fields.*`.
  - Gmail authentication or quota problems.

- **Sub-workflow reference:**  
  - None.

---

### Node: Send a message3

- **Type and technical role:**  
  `n8n-nodes-base.gmail`  
  Default/fallback-style acknowledgment email branch.

- **Configuration choices:**  
  - Recipient: `{{$json.fields.Email}}`
  - Subject: `Hi {{ $json.fields.Name }}, thank you for your inquiry!`
  - HTML body is similar to the support branch and references the submitted message and status `NEW`.

- **Key expressions or variables used:**  
  - `={{ $json.fields.Email }}`
  - `{{ $json.fields.Name }}`
  - `{{ $json.fields.Message }}`

- **Input and output connections:**  
  - Input: `Route by Category` fourth output
  - Output: none

- **Version-specific requirements:**  
  - Type version `2.2`
  - Requires Gmail credentials.

- **Edge cases or potential failure types:**  
  - The upstream Switch is configured for category `Default`, but AI returns `Other` or `Feedback`; this branch may never run unless routing is corrected.
  - Same field-path issues as the other Gmail nodes.

- **Sub-workflow reference:**  
  - None.

---

### Node: Send a message4

- **Type and technical role:**  
  `n8n-nodes-base.discord`  
  Sends an internal team alert to Discord after the sales email path.

- **Configuration choices:**  
  - Resource: `message`
  - Sends to selected guild and channel.
  - Content includes:
    - Name
    - Email
    - Lead Score
    - Priority
    - Message

- **Key expressions or variables used:**  
  - `{{$json.fields.Name}}`
  - `{{$json.fields.Email}}`
  - `{{$json.fields["Lead Score"]}}`
  - `{{$json.fields.Priority}}`
  - `{{$json.fields.Message}}`

- **Input and output connections:**  
  - Input: `Send a message`
  - Output: none

- **Version-specific requirements:**  
  - Type version `2`
  - Requires Discord credentials and selection of guild/channel resources.

- **Edge cases or potential failure types:**  
  - Likely data mismatch: the workflow never writes `Lead Score` or `Priority` into Airtable fields before this step.
  - If Discord permissions are insufficient for the target channel, sending will fail.
  - If the Gmail step fails, this alert will never run because it is downstream of the sales email node.
  - If the goal is “high-priority sales leads only,” there is no explicit priority check; all sales leads trigger this alert.

- **Sub-workflow reference:**  
  - None.

---

## 2.5 Documentation and In-Canvas Annotation Nodes

**Overview:**  
These sticky notes document the intended workflow design, setup requirements, and high-level phases. They do not affect execution but are important for human maintainers.

**Nodes Involved:**  
- Sticky Note6
- Sticky Note7
- Sticky Note8
- Sticky Note9
- Sticky Note10

### Node: Sticky Note6

- **Type and technical role:**  
  `n8n-nodes-base.stickyNote`  
  Visual documentation for the overall workflow.

- **Configuration choices:**  
  Contains a broad summary of purpose, setup notes, and expected Airtable fields.

- **Key expressions or variables used:**  
  - None

- **Input and output connections:**  
  - None

- **Version-specific requirements:**  
  - Type version `1`

- **Edge cases or potential failure types:**  
  - None operational.
  - Notes mention Airtable fields `Category`, `Lead Score`, and `Priority`, but the current workflow does not populate them.

- **Sub-workflow reference:**  
  - None.

---

### Node: Sticky Note7

- **Type and technical role:**  
  Sticky note documenting Step 1.

- **Configuration choices:**  
  Describes Typeform trigger and the sanitize/code phase.

- **Input and output connections:**  
  - None

- **Edge cases or potential failure types:**  
  - None operational.

---

### Node: Sticky Note8

- **Type and technical role:**  
  Sticky note documenting Step 2.

- **Configuration choices:**  
  Explains Airtable as the source of truth and fail-safe store.

- **Input and output connections:**  
  - None

- **Edge cases or potential failure types:**  
  - None operational.

---

### Node: Sticky Note9

- **Type and technical role:**  
  Sticky note documenting Step 3.

- **Configuration choices:**  
  Explains AI analysis and merge behavior.

- **Input and output connections:**  
  - None

- **Edge cases or potential failure types:**  
  - Mentions `GPT-4o-mini`, but actual model is `gpt-4.1-mini`.

---

### Node: Sticky Note10

- **Type and technical role:**  
  Sticky note documenting Step 4.

- **Configuration choices:**  
  Describes category-based routing, Gmail responses, and Discord alerts for sales leads.

- **Input and output connections:**  
  - None

- **Edge cases or potential failure types:**  
  - Notes mention Discord alerts for high-priority sales leads, but actual workflow sends Discord alerts for all sales-path leads.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Typeform Trigger | n8n-nodes-base.typeformTrigger | Receives new Typeform submissions |  | Sanitize Lead Data | **Lead intake and AI intelligence with Typeform, Airtable, and OpenAI**  / Captures new submissions from Typeform in real time / Cleans and standardizes lead data / Stores submissions in Airtable with status **NEW** / Uses OpenAI to classify leads, assign score, and set priority / Routes leads based on category / Sends email confirmations and Discord alerts for high-priority leads / Setup notes: Connect credentials: Typeform, Airtable, OpenAI, Gmail, Discord / Ensure Airtable fields: Name, Email, Message, Status, Category, Lead Score, Priority / Do not hardcode API keys—use n8n credentials / Use a Set node for configurable values if needed  / # Step 1 / Phase: Inbound & Cleaning / Typeform Trigger: Listens for the formId: UBdasOKD submission. / Sanitize Lead Data: Runs JavaScript to Title Case names, lowercase emails, and extract the domain for cleaner CRM entries. |
| Sanitize Lead Data | n8n-nodes-base.code | Normalizes raw form inputs | Typeform Trigger | Create a record | **Lead intake and AI intelligence with Typeform, Airtable, and OpenAI** / Captures new submissions from Typeform in real time / Cleans and standardizes lead data / Stores submissions in Airtable with status **NEW** / Uses OpenAI to classify leads, assign score, and set priority / Routes leads based on category / Sends email confirmations and Discord alerts for high-priority leads / Setup notes: Connect credentials: Typeform, Airtable, OpenAI, Gmail, Discord / Ensure Airtable fields: Name, Email, Message, Status, Category, Lead Score, Priority / Do not hardcode API keys—use n8n credentials / Use a Set node for configurable values if needed / # Step 1 / Phase: Inbound & Cleaning / Typeform Trigger: Listens for the formId: UBdasOKD submission. / Sanitize Lead Data: Runs JavaScript to Title Case names, lowercase emails, and extract the domain for cleaner CRM entries. |
| Create a record | n8n-nodes-base.airtable | Saves the cleaned lead in Airtable | Sanitize Lead Data | AI Lead Analyzer; Merge | **Lead intake and AI intelligence with Typeform, Airtable, and OpenAI** / Captures new submissions from Typeform in real time / Cleans and standardizes lead data / Stores submissions in Airtable with status **NEW** / Uses OpenAI to classify leads, assign score, and set priority / Routes leads based on category / Sends email confirmations and Discord alerts for high-priority leads / Setup notes: Connect credentials: Typeform, Airtable, OpenAI, Gmail, Discord / Ensure Airtable fields: Name, Email, Message, Status, Category, Lead Score, Priority / Do not hardcode API keys—use n8n credentials / Use a Set node for configurable values if needed / # Step 2 / Phase: The "Source of Truth" / Create a Record: Instantly pushes the sanitized data into Airtable. This ensures that even if the AI or Email steps fail later, you never lose the lead data. |
| AI Lead Analyzer | @n8n/n8n-nodes-langchain.openAi | Classifies lead intent and value using OpenAI | Create a record | Merge | **Lead intake and AI intelligence with Typeform, Airtable, and OpenAI** / Captures new submissions from Typeform in real time / Cleans and standardizes lead data / Stores submissions in Airtable with status **NEW** / Uses OpenAI to classify leads, assign score, and set priority / Routes leads based on category / Sends email confirmations and Discord alerts for high-priority leads / Setup notes: Connect credentials: Typeform, Airtable, OpenAI, Gmail, Discord / Ensure Airtable fields: Name, Email, Message, Status, Category, Lead Score, Priority / Do not hardcode API keys—use n8n credentials / Use a Set node for configurable values if needed / # Step 3 / Phase: AI Intelligence / AI Lead Analyzer: Passes the message to GPT-4o-mini. / Logic: The AI assigns a category, lead_score, and priority. / Merge: The workflow combines the original Airtable record data with the new AI insights to prepare for routing. |
| Merge | n8n-nodes-base.merge | Combines AI output and Airtable record | AI Lead Analyzer; Create a record | Route by Category | **Lead intake and AI intelligence with Typeform, Airtable, and OpenAI** / Captures new submissions from Typeform in real time / Cleans and standardizes lead data / Stores submissions in Airtable with status **NEW** / Uses OpenAI to classify leads, assign score, and set priority / Routes leads based on category / Sends email confirmations and Discord alerts for high-priority leads / Setup notes: Connect credentials: Typeform, Airtable, OpenAI, Gmail, Discord / Ensure Airtable fields: Name, Email, Message, Status, Category, Lead Score, Priority / Do not hardcode API keys—use n8n credentials / Use a Set node for configurable values if needed / # Step 3 / Phase: AI Intelligence / AI Lead Analyzer: Passes the message to GPT-4o-mini. / Logic: The AI assigns a category, lead_score, and priority. / Merge: The workflow combines the original Airtable record data with the new AI insights to prepare for routing. |
| Route by Category | n8n-nodes-base.switch | Routes leads to category-specific follow-up paths | Merge | Send a message; Send a message1; Send a message2; Send a message3 | **Lead intake and AI intelligence with Typeform, Airtable, and OpenAI** / Captures new submissions from Typeform in real time / Cleans and standardizes lead data / Stores submissions in Airtable with status **NEW** / Uses OpenAI to classify leads, assign score, and set priority / Routes leads based on category / Sends email confirmations and Discord alerts for high-priority leads / Setup notes: Connect credentials: Typeform, Airtable, OpenAI, Gmail, Discord / Ensure Airtable fields: Name, Email, Message, Status, Category, Lead Score, Priority / Do not hardcode API keys—use n8n credentials / Use a Set node for configurable values if needed / # Step 4 / Phase: Routing & Response / Route by Category: The Switch Node reads the AI's "Category" output. / Gmail (Branches): / Sales: Sends a high-energy "Let's Build" email. / Support/Partnership/Default: Sends tailored professional confirmations. / Discord Alert: A final notification is sent to the team (via the general channel) for Sales leads, including the AI-generated Lead Score and Priority. |
| Send a message | n8n-nodes-base.gmail | Sends the sales acknowledgment email | Route by Category | Send a message4 | **Lead intake and AI intelligence with Typeform, Airtable, and OpenAI** / Captures new submissions from Typeform in real time / Cleans and standardizes lead data / Stores submissions in Airtable with status **NEW** / Uses OpenAI to classify leads, assign score, and set priority / Routes leads based on category / Sends email confirmations and Discord alerts for high-priority leads / Setup notes: Connect credentials: Typeform, Airtable, OpenAI, Gmail, Discord / Ensure Airtable fields: Name, Email, Message, Status, Category, Lead Score, Priority / Do not hardcode API keys—use n8n credentials / Use a Set node for configurable values if needed / # Step 4 / Phase: Routing & Response / Route by Category: The Switch Node reads the AI's "Category" output. / Gmail (Branches): / Sales: Sends a high-energy "Let's Build" email. / Support/Partnership/Default: Sends tailored professional confirmations. / Discord Alert: A final notification is sent to the team (via the general channel) for Sales leads, including the AI-generated Lead Score and Priority. |
| Send a message1 | n8n-nodes-base.gmail | Sends the support acknowledgment email | Route by Category |  | **Lead intake and AI intelligence with Typeform, Airtable, and OpenAI** / Captures new submissions from Typeform in real time / Cleans and standardizes lead data / Stores submissions in Airtable with status **NEW** / Uses OpenAI to classify leads, assign score, and set priority / Routes leads based on category / Sends email confirmations and Discord alerts for high-priority leads / Setup notes: Connect credentials: Typeform, Airtable, OpenAI, Gmail, Discord / Ensure Airtable fields: Name, Email, Message, Status, Category, Lead Score, Priority / Do not hardcode API keys—use n8n credentials / Use a Set node for configurable values if needed / # Step 4 / Phase: Routing & Response / Route by Category: The Switch Node reads the AI's "Category" output. / Gmail (Branches): / Sales: Sends a high-energy "Let's Build" email. / Support/Partnership/Default: Sends tailored professional confirmations. / Discord Alert: A final notification is sent to the team (via the general channel) for Sales leads, including the AI-generated Lead Score and Priority. |
| Send a message2 | n8n-nodes-base.gmail | Sends the partnership acknowledgment email | Route by Category |  | **Lead intake and AI intelligence with Typeform, Airtable, and OpenAI** / Captures new submissions from Typeform in real time / Cleans and standardizes lead data / Stores submissions in Airtable with status **NEW** / Uses OpenAI to classify leads, assign score, and set priority / Routes leads based on category / Sends email confirmations and Discord alerts for high-priority leads / Setup notes: Connect credentials: Typeform, Airtable, OpenAI, Gmail, Discord / Ensure Airtable fields: Name, Email, Message, Status, Category, Lead Score, Priority / Do not hardcode API keys—use n8n credentials / Use a Set node for configurable values if needed / # Step 4 / Phase: Routing & Response / Route by Category: The Switch Node reads the AI's "Category" output. / Gmail (Branches): / Sales: Sends a high-energy "Let's Build" email. / Support/Partnership/Default: Sends tailored professional confirmations. / Discord Alert: A final notification is sent to the team (via the general channel) for Sales leads, including the AI-generated Lead Score and Priority. |
| Send a message3 | n8n-nodes-base.gmail | Sends the fallback/default acknowledgment email | Route by Category |  | **Lead intake and AI intelligence with Typeform, Airtable, and OpenAI** / Captures new submissions from Typeform in real time / Cleans and standardizes lead data / Stores submissions in Airtable with status **NEW** / Uses OpenAI to classify leads, assign score, and set priority / Routes leads based on category / Sends email confirmations and Discord alerts for high-priority leads / Setup notes: Connect credentials: Typeform, Airtable, OpenAI, Gmail, Discord / Ensure Airtable fields: Name, Email, Message, Status, Category, Lead Score, Priority / Do not hardcode API keys—use n8n credentials / Use a Set node for configurable values if needed / # Step 4 / Phase: Routing & Response / Route by Category: The Switch Node reads the AI's "Category" output. / Gmail (Branches): / Sales: Sends a high-energy "Let's Build" email. / Support/Partnership/Default: Sends tailored professional confirmations. / Discord Alert: A final notification is sent to the team (via the general channel) for Sales leads, including the AI-generated Lead Score and Priority. |
| Send a message4 | n8n-nodes-base.discord | Sends internal Discord alert for sales-path leads | Send a message |  | **Lead intake and AI intelligence with Typeform, Airtable, and OpenAI** / Captures new submissions from Typeform in real time / Cleans and standardizes lead data / Stores submissions in Airtable with status **NEW** / Uses OpenAI to classify leads, assign score, and set priority / Routes leads based on category / Sends email confirmations and Discord alerts for high-priority leads / Setup notes: Connect credentials: Typeform, Airtable, OpenAI, Gmail, Discord / Ensure Airtable fields: Name, Email, Message, Status, Category, Lead Score, Priority / Do not hardcode API keys—use n8n credentials / Use a Set node for configurable values if needed / # Step 4 / Phase: Routing & Response / Route by Category: The Switch Node reads the AI's "Category" output. / Gmail (Branches): / Sales: Sends a high-energy "Let's Build" email. / Support/Partnership/Default: Sends tailored professional confirmations. / Discord Alert: A final notification is sent to the team (via the general channel) for Sales leads, including the AI-generated Lead Score and Priority. |
| Sticky Note6 | n8n-nodes-base.stickyNote | Visual documentation for overall workflow |  |  |  |
| Sticky Note7 | n8n-nodes-base.stickyNote | Visual documentation for inbound/cleaning phase |  |  |  |
| Sticky Note8 | n8n-nodes-base.stickyNote | Visual documentation for Airtable storage phase |  |  |  |
| Sticky Note9 | n8n-nodes-base.stickyNote | Visual documentation for AI phase |  |  |  |
| Sticky Note10 | n8n-nodes-base.stickyNote | Visual documentation for routing/response phase |  |  |  |

---

## 4. Reproducing the Workflow from Scratch

Below is the exact build order to recreate the workflow manually in n8n.

### 1. Create the trigger node
1. Add a **Typeform Trigger** node.
2. Select or create **Typeform credentials**.
3. Set the **Form ID** to `UBdasOKD`.
4. Activate the trigger or test it with a sample submission.
5. Confirm the incoming payload includes the question labels:
   - `What is your full name?`
   - `What is your email address?`
   - `What project can we help you with today?`

### 2. Add the data-cleaning node
6. Add a **Code** node named **Sanitize Lead Data**.
7. Connect **Typeform Trigger → Sanitize Lead Data**.
8. Paste this logic conceptually:
   - read the three Typeform answer fields,
   - trim whitespace,
   - convert name to title case,
   - lowercase email,
   - extract the domain after `@`,
   - return a new JSON object with:
     - `Name`
     - `Email`
     - `Message`
     - `Domain`

9. Ensure the output contains these exact top-level keys:
   - `Name`
   - `Email`
   - `Message`
   - `Domain`

### 3. Prepare Airtable
10. In Airtable, create or identify a base and table for leads.
11. Ensure the table contains at minimum:
   - `Name` as text
   - `Email` as text/email
   - `Message` as long text
   - `Status` as single select with at least:
     - `New`
     - `Done`

12. If you want the routing logic to work exactly as intended, also add:
   - `Category`
   - `Lead Score`
   - `Priority`
   
   Note: the provided workflow does not actually populate these fields, but later nodes assume they exist.

### 4. Add Airtable storage
13. Add an **Airtable** node named **Create a record**.
14. Connect **Sanitize Lead Data → Create a record**.
15. Configure:
   - **Operation:** Create
   - **Base:** select your target base
   - **Table:** select your target table
16. Map fields:
   - `Name` = `{{$json.Name}}`
   - `Email` = `{{$json.Email}}`
   - `Message` = `{{$json.Message}}`
   - `Status` = `New`

### 5. Add AI classification
17. Add an **OpenAI** node from the LangChain/OpenAI integration named **AI Lead Analyzer**.
18. Connect **Create a record → AI Lead Analyzer**.
19. Configure **OpenAI credentials**.
20. Select model **gpt-4.1-mini**.
21. In the response/prompt field, instruct the model to:
   - analyze `{{$json["Message"]}}`,
   - classify into `Sales`, `Support`, `Partnership`, `Feedback`, or `Other`,
   - assign `lead_score` from 1 to 10,
   - assign `priority` from `Low`, `Medium`, `High`,
   - return strict JSON only with:
     - `category`
     - `lead_score`
     - `priority`

22. Keep the prompt strict to reduce malformed responses.

### 6. Add the merge node
23. Add a **Merge** node named **Merge**.
24. Connect:
   - **AI Lead Analyzer → Merge** on input 0
   - **Create a record → Merge** on input 1
25. Configure:
   - **Mode:** Combine
   - **Combine By:** Position

### 7. Add routing
26. Add a **Switch** node named **Route by Category**.
27. Connect **Merge → Route by Category**.
28. Create four outputs with conditions:
   - Output 1: Sales
   - Output 2: Support
   - Output 3: Partnership
   - Output 4: Default

29. In the provided workflow, the compared field is `{{$json.fields.Category}}`.

### 8. Important correction before production
30. The original workflow has a structural mismatch:
   - OpenAI returns `category`
   - Switch expects `fields.Category`
31. To make the workflow reliable, add an intermediate **Set** or **Code** node after the merge that maps data into a consistent structure, for example:
   - `fields.Name`
   - `fields.Email`
   - `fields.Message`
   - `fields.Category`
   - `fields["Lead Score"]`
   - `fields.Priority`

32. Alternatively, adjust all downstream expressions to use the actual merged structure instead of `fields.*`.

### 9. Add sales email branch
33. Add a **Gmail** node named **Send a message**.
34. Connect **Route by Category output 0 → Send a message**.
35. Configure **Gmail OAuth2 credentials** with send permission.
36. Set:
   - **To:** `{{$json.fields.Email}}`
   - **Subject:** `🔥 Let's Build Something Great, {{$json.fields.Name}}`
   - **Body:** a friendly sales confirmation message
37. Replace placeholder `[Your Services Link]` with a real URL.

### 10. Add support email branch
38. Add a **Gmail** node named **Send a message1**.
39. Connect **Route by Category output 1 → Send a message1**.
40. Configure:
   - **To:** `{{$json.fields.Email}}`
   - **Subject:** `We Got Your Request 👍`
   - **Body:** HTML acknowledgment including:
     - name,
     - quoted original message,
     - note that status is `NEW`.

### 11. Add partnership email branch
41. Add a **Gmail** node named **Send a message2**.
42. Connect **Route by Category output 2 → Send a message2**.
43. Configure:
   - **To:** `{{$json.fields.Email}}`
   - **Subject:** `Partnership Opportunity 🚀`
   - **Body:** short collaboration-oriented response.

### 12. Add fallback/default email branch
44. Add a **Gmail** node named **Send a message3**.
45. Connect **Route by Category output 3 → Send a message3**.
46. Configure:
   - **To:** `{{$json.fields.Email}}`
   - **Subject:** `Hi {{ $json.fields.Name }}, thank you for your inquiry!`
   - **Body:** generic acknowledgment.

### 13. Add Discord alert for sales path
47. Add a **Discord** node named **Send a message4**.
48. Connect **Send a message → Send a message4**.
49. Configure **Discord credentials**.
50. Select the target **Guild** and **Channel**.
51. Set content to include:
   - Name
   - Email
   - Lead Score
   - Priority
   - Message

### 14. Add canvas documentation notes
52. Add five **Sticky Note** nodes if you want the same visual structure:
   - one overview note,
   - one note for inbound/cleaning,
   - one note for Airtable,
   - one note for AI,
   - one note for routing/response.

### 15. Credential setup summary
53. Configure these credentials in n8n:
   - **Typeform**
   - **Airtable**
   - **OpenAI**
   - **Gmail OAuth2**
   - **Discord**

### 16. Recommended fixes to fully reproduce the intended behavior
54. Add a node after OpenAI to parse or normalize the AI result if needed.
55. Add a node to map:
   - `category` → `fields.Category`
   - `lead_score` → `fields["Lead Score"]`
   - `priority` → `fields.Priority`
56. Optionally update the Airtable record after AI classification so Airtable truly stores:
   - Category
   - Lead Score
   - Priority
57. Change the Switch fallback rule from `Default` to either:
   - `Other`, or
   - create separate branches for `Feedback` and `Other`.
58. If Discord should only fire for high-priority sales leads, add an **IF** node after the sales Gmail step checking priority before Discord.
59. Test each category with sample messages:
   - Sales inquiry
   - Support issue
   - Partnership proposal
   - Feedback comment
   - Empty/spam message

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Lead intake and AI intelligence with Typeform, Airtable, and OpenAI | Overall workflow note |
| Captures new submissions from Typeform in real time | Overall workflow note |
| Cleans and standardizes lead data | Overall workflow note |
| Stores submissions in Airtable with status **NEW** | Overall workflow note |
| Uses OpenAI to classify leads, assign score, and set priority | Overall workflow note |
| Routes leads based on category | Overall workflow note |
| Sends email confirmations and Discord alerts for high-priority leads | Overall workflow note |
| Connect credentials: Typeform, Airtable, OpenAI, Gmail, Discord | Setup note |
| Ensure Airtable fields: Name, Email, Message, Status, Category, Lead Score, Priority | Setup note |
| Do not hardcode API keys—use n8n credentials | Setup note |
| Use a Set node for configurable values if needed | Setup note |
| Step 1: Inbound & Cleaning | Sticky note context |
| Typeform Trigger listens for formId `UBdasOKD` | Sticky note context |
| Sanitize Lead Data title-cases names, lowercases emails, and extracts domain | Sticky note context |
| Step 2: The "Source of Truth" | Sticky note context |
| Create a Record instantly pushes sanitized data into Airtable so leads are not lost if AI or email steps fail | Sticky note context |
| Step 3: AI Intelligence | Sticky note context |
| AI Lead Analyzer passes the message to GPT-4o-mini | Sticky note context; note this differs from actual configured model `gpt-4.1-mini` |
| Merge combines Airtable record data with AI insights to prepare for routing | Sticky note context |
| Step 4: Routing & Response | Sticky note context |
| Route by Category reads the AI’s category output | Sticky note context |
| Sales sends a high-energy “Let’s Build” email | Sticky note context |
| Support/Partnership/Default send tailored professional confirmations | Sticky note context |
| Discord alert is sent to the team for Sales leads with Lead Score and Priority | Sticky note context |
| Placeholder text `[Your Services Link]` should be replaced with a real URL before use | Gmail sales message content |

## Operational observations

- The workflow intention is clear and strong, but the current implementation likely has field-structure mismatches.
- Most downstream nodes reference `{{$json.fields.*}}`, while the AI node likely returns values outside the Airtable `fields` object.
- The fallback switch rule uses `Default`, but the AI prompt defines `Feedback` and `Other`.
- The workflow description says Discord is for high-priority leads, but no priority filter is implemented.
- Airtable is treated as the source of truth, yet AI results are not written back into Airtable in the current configuration.

If you want, I can also provide:
1. a **corrected field mapping design**, or  
2. a **production-safe revised n8n version** of this workflow logic in plain English.