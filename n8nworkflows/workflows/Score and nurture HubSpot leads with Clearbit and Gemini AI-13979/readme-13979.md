Score and nurture HubSpot leads with Clearbit and Gemini AI

https://n8nworkflows.xyz/workflows/score-and-nurture-hubspot-leads-with-clearbit-and-gemini-ai-13979


# Score and nurture HubSpot leads with Clearbit and Gemini AI

# 1. Workflow Overview

This workflow automatically evaluates newly created HubSpot contacts, enriches them with Clearbit company data, scores them using both deterministic rules and Gemini AI, then routes them into either immediate sales outreach or a nurturing path. It also notifies the sales team for high-priority leads and logs scoring activity to Google Sheets.

Typical use cases:
- Prioritizing inbound B2B leads automatically
- Generating personalized outreach for high-value prospects
- Feeding lower-priority leads into a nurture process
- Keeping a historical scoring log for later analysis

## 1.1 Trigger and Enrichment

The workflow starts when a new HubSpot contact is created. It then calls Clearbit to enrich that contact with company-level context such as industry, employee count, and funding data.

## 1.2 Rules-Based Lead Scoring

After enrichment, a Set node defines static scoring criteria such as target industries, company size weights, role weights, and threshold values. A Code node uses those criteria together with HubSpot and Clearbit data to calculate a base score.

## 1.3 AI Analysis and Final Classification

The workflow sends the enriched lead context and basic score to a Gemini-powered LLM chain. The AI is instructed to produce a JSON response containing an adjusted score, final score, category, reasoning, personalized email content, and nurture recommendations.

## 1.4 Routing and CRM Actions

An If node checks whether the AI-generated final score meets the hot-lead threshold. Based on that result, the workflow updates the HubSpot contact and either sends immediate outreach / sales notification or places the lead into a nurture path.

## 1.5 Notifications and Logging

Hot leads trigger a Slack alert and a Gmail message. Any lead that passes through the HubSpot update step is then logged in Google Sheets for tracking and analysis.

---

# 2. Block-by-Block Analysis

## Block 1: Trigger and Contact Enrichment

### Overview
This block receives newly created HubSpot contacts and enriches them using Clearbit. It prepares the dataset needed for downstream scoring and AI analysis.

### Nodes Involved
- HubSpot Contact Created
- Clearbit Company Enrichment

### Node Details

#### 1. HubSpot Contact Created
- **Type and technical role:** `n8n-nodes-base.hubspotTrigger`  
  Event trigger node that starts the workflow when a HubSpot contact is created.
- **Configuration choices:**
  - Trigger is configured for HubSpot events.
  - No additional fields are configured.
- **Key expressions or variables used:**
  - Downstream nodes reference this node using expressions like:
    - `$('HubSpot Contact Created').first().json.properties.email`
    - `$('HubSpot Contact Created').first().json.properties.jobtitle`
    - `$('HubSpot Contact Created').first().json.properties.hs_object_id`
- **Input and output connections:**
  - No input; this is an entry point.
  - Output goes to **Clearbit Company Enrichment**.
- **Version-specific requirements:**
  - `typeVersion: 1`
  - Requires supported HubSpot trigger credentials and webhook/event registration in n8n.
- **Edge cases or potential failure types:**
  - HubSpot credential/authentication failures
  - Trigger registration issues
  - Contact objects missing expected properties such as `firstname`, `lastname`, `email`, or `jobtitle`
  - Differences in HubSpot property naming across portals
- **Sub-workflow reference:** None

#### 2. Clearbit Company Enrichment
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Performs an HTTP request to Clearbit’s person enrichment endpoint.
- **Configuration choices:**
  - URL: `https://person.clearbit.com/v2/people/find`
  - Authentication: generic credential type using HTTP header auth
  - No explicit query parameters are shown in the JSON
- **Key expressions or variables used:**
  - No request-time expressions are configured in the visible parameters.
  - Downstream nodes expect a Clearbit-style response under:
    - `company.name`
    - `company.category.industry`
    - `company.metrics.employees`
    - `company.metrics.raised`
    - `company.domain`
- **Input and output connections:**
  - Input from **HubSpot Contact Created**
  - Output to **Set Scoring Criteria**
- **Version-specific requirements:**
  - `typeVersion: 4.2`
  - Requires a valid HTTP Header Auth credential containing the Clearbit API key.
- **Edge cases or potential failure types:**
  - As configured, the node does not visibly include email or domain query parameters; without them, Clearbit may return an error or no match.
  - API auth failure
  - Rate limiting from Clearbit
  - No enrichment found
  - Partial enrichment with missing `company` object
- **Sub-workflow reference:** None

---

## Block 2: Static Criteria and Basic Score Calculation

### Overview
This block defines the deterministic scoring rules and computes a base lead score from enriched company data and HubSpot contact fields. It creates a normalized payload for AI analysis.

### Nodes Involved
- Set Scoring Criteria
- Calculate Basic Score

### Node Details

#### 3. Set Scoring Criteria
- **Type and technical role:** `n8n-nodes-base.set`  
  Adds static configuration values to the workflow item.
- **Configuration choices:**
  - Creates the following fields:
    - `target_industries`: `['SaaS', 'Technology', 'Fintech', 'E-commerce']`
    - `company_size_weights`: startup 3, small 5, medium 8, large 10
    - `role_weights`: founder 10, ceo 10, cto 9, vp 8, director 7, manager 6, other 3
    - `hot_threshold`: 80
    - `warm_threshold`: 50
- **Key expressions or variables used:**
  - These values are consumed later by:
    - **Calculate Basic Score**
    - **Score Threshold Check**
- **Input and output connections:**
  - Input from **Clearbit Company Enrichment**
  - Output to **Calculate Basic Score**
- **Version-specific requirements:**
  - `typeVersion: 3.4`
- **Edge cases or potential failure types:**
  - If array/object values are incorrectly interpreted as strings in some environments, the Code node may fail or behave unexpectedly.
  - Thresholds can become inconsistent with AI prompt logic if changed here but not reflected conceptually elsewhere.
- **Sub-workflow reference:** None

#### 4. Calculate Basic Score
- **Type and technical role:** `n8n-nodes-base.code`  
  Runs custom JavaScript to calculate a rules-based lead score.
- **Configuration choices:**
  - Pulls source data from:
    - `HubSpot Contact Created`
    - `Clearbit Company Enrichment`
    - current item from **Set Scoring Criteria**
  - Computes:
    - `basic_score`
    - `size_category`
    - `industry_match`
    - plus carries forward `contact_data` and `enrichment_data`
- **Key expressions or variables used:**
  - `$('HubSpot Contact Created').first().json`
  - `$('Clearbit Company Enrichment').first().json`
  - `$json`
  - Employee-based size buckets:
    - `>1000 => large`
    - `>100 => medium`
    - `>10 => small`
    - otherwise `startup`
  - Industry match:
    - `criteria.target_industries.some(...)`
  - Role matching from HubSpot `jobtitle`
  - Funding bonus if `enrichment.company.metrics.raised` exists
- **Input and output connections:**
  - Input from **Set Scoring Criteria**
  - Output to **AI Lead Analysis**
- **Version-specific requirements:**
  - `typeVersion: 2`
  - Requires Code node JavaScript support in the n8n instance.
- **Edge cases or potential failure types:**
  - Missing `properties.jobtitle` can reduce role scoring to default
  - Missing Clearbit `company` or nested metrics/category data can affect scoring, though optional chaining is used in several places
  - If `target_industries` or `role_weights` are not proper array/object values, methods like `.some()` or `Object.entries()` may fail
  - The code assumes the first item from trigger and enrichment is the correct context; if batching is introduced later, this logic may need redesign
- **Sub-workflow reference:** None

---

## Block 3: Gemini AI Scoring and Content Generation

### Overview
This block sends the lead profile to Gemini and asks for a machine-readable analysis. The output is expected to include a final score, category, rationale, personalized email content, and nurture guidance.

### Nodes Involved
- Google Gemini Chat Model
- AI Lead Analysis

### Node Details

#### 5. Google Gemini Chat Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatGoogleGemini`  
  Provides the language model backend used by the LangChain LLM chain node.
- **Configuration choices:**
  - Uses Google Gemini / PaLM API credentials
  - No advanced options are explicitly set
- **Key expressions or variables used:**
  - None directly in the node
- **Input and output connections:**
  - AI language model connection into **AI Lead Analysis**
  - No standard main input/output path
- **Version-specific requirements:**
  - `typeVersion: 1`
  - Requires compatible Google Gemini credentials in n8n
  - Availability depends on installed n8n LangChain AI nodes
- **Edge cases or potential failure types:**
  - Invalid or expired Gemini credentials
  - Quota exhaustion
  - Regional/model availability issues
  - Output variability depending on model behavior
- **Sub-workflow reference:** None

#### 6. AI Lead Analysis
- **Type and technical role:** `@n8n/n8n-nodes-langchain.chainLlm`  
  Prompts the Gemini model to analyze the lead and return structured JSON.
- **Configuration choices:**
  - Prompt type: defined directly in the node
  - Prompt includes:
    - contact name, email, title
    - company name, industry, employee count, funding, domain
    - `basic_score`, `industry_match`, `size_category`
  - The model is instructed to return JSON with:
    - `ai_score_adjustment`
    - `final_score`
    - `lead_category`
    - `reasoning`
    - `personalized_email.subject`
    - `personalized_email.body`
    - `nurturing_notes`
- **Key expressions or variables used:**
  - `{{ $json.contact_data.properties.firstname }}`
  - `{{ $json.enrichment_data.company.name }}`
  - `{{ $json.basic_score }}`
  - and similar nested expressions for enrichment and contact data
- **Input and output connections:**
  - Main input from **Calculate Basic Score**
  - AI model input from **Google Gemini Chat Model**
  - Main output to **Score Threshold Check**
- **Version-specific requirements:**
  - `typeVersion: 1.4`
  - Requires LangChain-compatible AI nodes installed and configured in the n8n instance
- **Edge cases or potential failure types:**
  - Model may return non-JSON or malformed JSON
  - Missing enrichment fields may weaken prompt quality or create blank values
  - If the node does not parse model output into structured JSON automatically, downstream expressions like `$json.final_score` may fail
  - Large prompt or transient model errors can cause timeouts
- **Sub-workflow reference:** None

---

## Block 4: Lead Routing and CRM Updates

### Overview
This block evaluates whether the lead qualifies as hot and routes it accordingly. It also updates HubSpot regardless of branch, though the two HubSpot update nodes likely need separate field mappings to be truly useful.

### Nodes Involved
- Score Threshold Check
- Update Contact Properties
- Add to Nurturing Sequence

### Node Details

#### 7. Score Threshold Check
- **Type and technical role:** `n8n-nodes-base.if`  
  Branching node that checks whether the final AI score meets the hot-lead threshold.
- **Configuration choices:**
  - Condition:
    - Left value: `{{ $json.final_score }}`
    - Operator: number greater than or equal
    - Right value: `{{ $('Set Scoring Criteria').first().json.hot_threshold }}`
  - True branch = hot lead
  - False branch = non-hot lead
- **Key expressions or variables used:**
  - `$json.final_score`
  - `$('Set Scoring Criteria').first().json.hot_threshold`
- **Input and output connections:**
  - Input from **AI Lead Analysis**
  - True output to:
    - **Update Contact Properties**
    - **Send Personalized Email**
    - **Notify Sales Team**
  - False output to:
    - **Update Contact Properties**
    - **Add to Nurturing Sequence**
- **Version-specific requirements:**
  - `typeVersion: 2`
- **Edge cases or potential failure types:**
  - If `final_score` is missing or non-numeric, strict number validation may fail
  - Warm threshold exists in the Set node but is not used here
  - All non-hot leads are treated the same; no dedicated warm vs cold branching is implemented
- **Sub-workflow reference:** None

#### 8. Update Contact Properties
- **Type and technical role:** `n8n-nodes-base.hubspot`  
  Updates a HubSpot contact after scoring.
- **Configuration choices:**
  - Operation: `update`
  - No detailed field mapping is visible in the JSON
- **Key expressions or variables used:**
  - None visible in parameters, but this node should typically use contact ID and fields such as score/category.
- **Input and output connections:**
  - Input from both branches of **Score Threshold Check**
  - Output to **Log Scoring History**
- **Version-specific requirements:**
  - `typeVersion: 2`
  - Requires HubSpot credentials with update permissions
- **Edge cases or potential failure types:**
  - As shown, missing update-field configuration may cause execution failure or no useful CRM update
  - Contact ID may need explicit mapping, likely from `hs_object_id`
  - If both branches are active over time, configuration must be branch-safe
- **Sub-workflow reference:** None

#### 9. Add to Nurturing Sequence
- **Type and technical role:** `n8n-nodes-base.hubspot`  
  Intended to update the contact for a nurture sequence path.
- **Configuration choices:**
  - Operation: `update`
  - No detailed field mapping is visible
- **Key expressions or variables used:**
  - None visible, but it likely should set lifecycle stage, list membership, sequence flag, or custom properties.
- **Input and output connections:**
  - Input from the false branch of **Score Threshold Check**
  - No output connection configured
- **Version-specific requirements:**
  - `typeVersion: 2`
  - Requires HubSpot credentials with contact update permissions
- **Edge cases or potential failure types:**
  - Incomplete configuration may make the node ineffective
  - No separate warm/cold handling despite AI output containing `lead_category`
- **Sub-workflow reference:** None

---

## Block 5: Outreach, Notifications, and Logging

### Overview
This block performs hot-lead engagement and recordkeeping. It sends an email, alerts sales in Slack, and writes a lead-scoring history row into Google Sheets.

### Nodes Involved
- Send Personalized Email
- Notify Sales Team
- Log Scoring History

### Node Details

#### 10. Send Personalized Email
- **Type and technical role:** `n8n-nodes-base.gmail`  
  Sends an email to the lead using AI-generated subject and body.
- **Configuration choices:**
  - Recipient:
    - `={{ $('HubSpot Contact Created').first().json.properties.email }}`
  - Subject:
    - `={{ $json.personalized_email.subject }}`
  - Message:
    - `={{ $json.personalized_email.body }}`
  - BCC:
    - `user@example.com`
- **Key expressions or variables used:**
  - Contact email from HubSpot trigger
  - `personalized_email.subject`
  - `personalized_email.body`
- **Input and output connections:**
  - Input from true branch of **Score Threshold Check**
  - No output connection configured
- **Version-specific requirements:**
  - `typeVersion: 2.1`
  - Requires Gmail credentials and a connected sender account
- **Edge cases or potential failure types:**
  - AI output missing `personalized_email`
  - Gmail auth failure
  - Sending restrictions, quota issues, or spam-related limitations
  - BCC should be replaced from placeholder value before production use
- **Sub-workflow reference:** None

#### 11. Notify Sales Team
- **Type and technical role:** `n8n-nodes-base.slack`  
  Sends a hot-lead alert to a Slack channel.
- **Configuration choices:**
  - Sends formatted text to channel named `sales-alerts`
  - Message includes contact details, company, score, category, and reasoning
- **Key expressions or variables used:**
  - HubSpot contact fields:
    - first name, last name, title, email, `hs_object_id`
  - AI output:
    - `final_score`
    - `lead_category`
    - `reasoning`
  - Enrichment:
    - `enrichment_data?.company?.name`
- **Input and output connections:**
  - Input from true branch of **Score Threshold Check**
  - No output connection configured
- **Version-specific requirements:**
  - `typeVersion: 2.1`
  - Requires Slack credentials and channel access
- **Edge cases or potential failure types:**
  - `hs_object_id` is inserted as if it were a direct HubSpot URL, but it is only an object ID; the resulting Slack link will not be a valid HubSpot record URL unless changed
  - Slack channel name must exist and be accessible to the app
  - Missing contact/company fields can make the message incomplete
- **Sub-workflow reference:** None

#### 12. Log Scoring History
- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Appends or updates a row in Google Sheets to keep a scoring history.
- **Configuration choices:**
  - Operation: `appendOrUpdate`
  - Sheet name: `Lead Scoring Log`
  - Document ID: placeholder `your-lead-scoring-sheet-id`
  - Column mapping includes:
    - Contact Name
    - Email
    - Company
    - Job Title
    - Industry
    - Employee Count
    - Basic Score
    - Final Score
    - Category
    - Reasoning
    - Timestamp
- **Key expressions or variables used:**
  - HubSpot contact fields from trigger node
  - Enrichment fields from AI/current payload
  - `$('Calculate Basic Score').first().json.basic_score`
  - `new Date().toISOString()`
- **Input and output connections:**
  - Input from **Update Contact Properties**
  - No output connection configured
- **Version-specific requirements:**
  - `typeVersion: 4.4`
  - Requires Google Sheets credentials and access to the target spreadsheet
- **Edge cases or potential failure types:**
  - Spreadsheet ID is a placeholder and must be replaced
  - `appendOrUpdate` generally requires matching logic or key fields depending on node configuration; if not fully configured, behavior may not match expectations
  - Missing values in AI or enrichment data may leave blank cells
- **Sub-workflow reference:** None

---

## Block 6: Documentation / Sticky Notes

### Overview
These nodes do not affect execution. They document the workflow’s purpose, setup, and logical grouping inside the n8n canvas.

### Nodes Involved
- Sticky Note
- Sticky Note1
- Sticky Note2
- Sticky Note3
- Sticky Note4

### Node Details

#### 13. Sticky Note
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Canvas documentation for the overall workflow.
- **Configuration choices:**
  - Contains workflow summary, setup steps, and customization ideas.
- **Input and output connections:**
  - None
- **Version-specific requirements:**
  - `typeVersion: 1`
- **Edge cases or potential failure types:**
  - None; non-executable
- **Sub-workflow reference:** None

#### 14. Sticky Note1
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:**
  - Documents the data enrichment section.
- **Input and output connections:**
  - None
- **Version-specific requirements:**
  - `typeVersion: 1`
- **Edge cases or potential failure types:**
  - None
- **Sub-workflow reference:** None

#### 15. Sticky Note2
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:**
  - Documents the AI-powered scoring section.
- **Input and output connections:**
  - None
- **Version-specific requirements:**
  - `typeVersion: 1`
- **Edge cases or potential failure types:**
  - None
- **Sub-workflow reference:** None

#### 16. Sticky Note3
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:**
  - Documents the lead routing and outreach section.
- **Input and output connections:**
  - None
- **Version-specific requirements:**
  - `typeVersion: 1`
- **Edge cases or potential failure types:**
  - None
- **Sub-workflow reference:** None

#### 17. Sticky Note4
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:**
  - Documents the notifications and tracking section.
- **Input and output connections:**
  - None
- **Version-specific requirements:**
  - `typeVersion: 1`
- **Edge cases or potential failure types:**
  - None
- **Sub-workflow reference:** None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| HubSpot Contact Created | HubSpot Trigger | Entry point when a new HubSpot contact is created |  | Clearbit Company Enrichment | # Lead Scoring Automation<br>This workflow automatically scores and nurtures new HubSpot contacts using Clearbit enrichment and Gemini AI analysis.<br>## How it works<br>1. **Trigger**: New contact created in HubSpot<br>2. **Enrich**: Clearbit adds company data<br>3. **Score**: Basic algorithm + AI enhancement<br>4. **Route**: Hot leads get immediate outreach, warm leads enter nurturing<br>5. **Notify**: Sales team alerted for hot leads<br>6. **Track**: All scores logged to Google Sheets<br>## Setup steps<br>1. Configure HubSpot, Clearbit, Gemini AI credentials<br>2. Update Slack channel name and Gmail settings<br>3. Create Google Sheet for scoring history<br>4. Adjust scoring thresholds in Set node<br>5. Customize email templates in AI prompt<br>## Customization<br>- Modify scoring criteria (industries, role weights)<br>- Adjust hot/warm thresholds (default: 80/50)<br>- Personalize email templates and Slack messages |
| Clearbit Company Enrichment | HTTP Request | Enrich lead/company data from Clearbit | HubSpot Contact Created | Set Scoring Criteria | ## Data enrichment<br>Clearbit adds company information to score based on size, industry, and funding status. |
| Set Scoring Criteria | Set | Defines scoring rules and thresholds | Clearbit Company Enrichment | Calculate Basic Score | ## Data enrichment<br>Clearbit adds company information to score based on size, industry, and funding status. |
| Calculate Basic Score | Code | Computes deterministic base score | Set Scoring Criteria | AI Lead Analysis | ## AI-powered scoring<br>Gemini AI analyzes the enriched data and generates personalized outreach content. |
| Google Gemini Chat Model | Google Gemini Chat Model | Supplies the LLM backend for AI analysis |  | AI Lead Analysis (AI language model connection) | ## AI-powered scoring<br>Gemini AI analyzes the enriched data and generates personalized outreach content. |
| AI Lead Analysis | LangChain LLM Chain | Produces AI-enhanced score and outreach JSON | Calculate Basic Score; Google Gemini Chat Model | Score Threshold Check | ## AI-powered scoring<br>Gemini AI analyzes the enriched data and generates personalized outreach content. |
| Score Threshold Check | If | Routes leads by final score threshold | AI Lead Analysis | Update Contact Properties; Send Personalized Email; Notify Sales Team; Add to Nurturing Sequence | ## Lead routing & outreach<br>Hot leads get personalized emails, warm leads enter nurturing sequences. |
| Update Contact Properties | HubSpot | Updates HubSpot contact after scoring | Score Threshold Check | Log Scoring History | ## Lead routing & outreach<br>Hot leads get personalized emails, warm leads enter nurturing sequences. |
| Add to Nurturing Sequence | HubSpot | Updates non-hot leads for nurture handling | Score Threshold Check |  | ## Lead routing & outreach<br>Hot leads get personalized emails, warm leads enter nurturing sequences. |
| Send Personalized Email | Gmail | Sends AI-generated outreach to hot leads | Score Threshold Check |  | ## Notifications & tracking<br>Sales team gets Slack alerts for hot leads. All scores are logged for analysis. |
| Notify Sales Team | Slack | Posts hot lead alert to Slack | Score Threshold Check |  | ## Notifications & tracking<br>Sales team gets Slack alerts for hot leads. All scores are logged for analysis. |
| Log Scoring History | Google Sheets | Records scoring result in spreadsheet | Update Contact Properties |  | ## Notifications & tracking<br>Sales team gets Slack alerts for hot leads. All scores are logged for analysis. |
| Sticky Note | Sticky Note | Canvas documentation |  |  |  |
| Sticky Note1 | Sticky Note | Canvas documentation |  |  |  |
| Sticky Note2 | Sticky Note | Canvas documentation |  |  |  |
| Sticky Note3 | Sticky Note | Canvas documentation |  |  |  |
| Sticky Note4 | Sticky Note | Canvas documentation |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it:  
   **Score and nurture HubSpot leads with Clearbit and Gemini AI**

2. **Add a HubSpot Trigger node**
   - Node type: **HubSpot Trigger**
   - Name it: **HubSpot Contact Created**
   - Configure HubSpot credentials.
   - Set it to trigger when a new contact is created.
   - Keep additional fields empty unless you need extra properties.

3. **Add an HTTP Request node for Clearbit**
   - Node type: **HTTP Request**
   - Name it: **Clearbit Company Enrichment**
   - Method: use the appropriate method required by Clearbit person lookup, typically `GET`
   - URL: `https://person.clearbit.com/v2/people/find`
   - Authentication: **Generic Credential Type**
   - Generic auth type: **HTTP Header Auth**
   - Create credentials that send the Clearbit API key as an Authorization header or according to Clearbit’s required header format.
   - Important: add query parameters such as the HubSpot contact email, otherwise Clearbit cannot reliably identify the person. Example concept:
     - `email = {{$json.properties.email}}`
   - Connect **HubSpot Contact Created → Clearbit Company Enrichment**

4. **Add a Set node for scoring constants**
   - Node type: **Set**
   - Name it: **Set Scoring Criteria**
   - Add the following fields:
     - `target_industries` as an array:
       - SaaS
       - Technology
       - Fintech
       - E-commerce
     - `company_size_weights` as an object:
       - startup: 3
       - small: 5
       - medium: 8
       - large: 10
     - `role_weights` as an object:
       - founder: 10
       - ceo: 10
       - cto: 9
       - vp: 8
       - director: 7
       - manager: 6
       - other: 3
     - `hot_threshold` as number: `80`
     - `warm_threshold` as number: `50`
   - Connect **Clearbit Company Enrichment → Set Scoring Criteria**

5. **Add a Code node**
   - Node type: **Code**
   - Name it: **Calculate Basic Score**
   - Paste JavaScript equivalent to the workflow logic:
     - Read contact data from `HubSpot Contact Created`
     - Read enrichment from `Clearbit Company Enrichment`
     - Read criteria from current item
     - Calculate:
       - size category from employee count
       - score boost for target industry match
       - score boost from job title / role
       - funding bonus
     - Return:
       - `contact_data`
       - `enrichment_data`
       - `basic_score`
       - `size_category`
       - `industry_match`
   - Connect **Set Scoring Criteria → Calculate Basic Score**

6. **Add a Google Gemini Chat Model node**
   - Node type: **Google Gemini Chat Model**
   - Name it: **Google Gemini Chat Model**
   - Configure Google Gemini / PaLM API credentials.
   - Default options can remain unchanged unless you want to set model or temperature explicitly.

7. **Add an AI chain node**
   - Node type: **Basic LLM Chain / LangChain LLM Chain**
   - Name it: **AI Lead Analysis**
   - Set prompt type to define the text manually.
   - Use a prompt that includes:
     - Contact name
     - Email
     - Job title
     - Company
     - Industry
     - Employee count
     - Funding
     - Website
     - Basic score
     - Industry match
     - Company size
   - Instruct the model to respond in JSON with exactly these fields:
     - `ai_score_adjustment`
     - `final_score`
     - `lead_category`
     - `reasoning`
     - `personalized_email.subject`
     - `personalized_email.body`
     - `nurturing_notes`
   - Connect:
     - **Calculate Basic Score → AI Lead Analysis**
     - **Google Gemini Chat Model → AI Lead Analysis** via the AI language model port

8. **Add an If node**
   - Node type: **If**
   - Name it: **Score Threshold Check**
   - Set a numeric condition:
     - Left value: `{{$json.final_score}}`
     - Operation: greater than or equal
     - Right value: `{{$('Set Scoring Criteria').first().json.hot_threshold}}`
   - Connect **AI Lead Analysis → Score Threshold Check**

9. **Add a HubSpot node for CRM updates**
   - Node type: **HubSpot**
   - Name it: **Update Contact Properties**
   - Operation: **Update**
   - Configure HubSpot credentials.
   - Use the HubSpot contact ID from trigger data, typically `hs_object_id`.
   - Map useful fields such as:
     - lead score
     - AI final score
     - lead category
     - reasoning
     - last scored timestamp
   - Connect this node from both outputs of **Score Threshold Check**.

10. **Add a second HubSpot node for nurture handling**
    - Node type: **HubSpot**
    - Name it: **Add to Nurturing Sequence**
    - Operation: **Update**
    - Configure the same HubSpot credentials.
    - Use the same contact ID mapping.
    - Set one or more nurture-related properties, for example:
      - lifecycle stage
      - lead status
      - nurture flag
      - campaign/list membership marker
    - Connect the **false** branch of **Score Threshold Check → Add to Nurturing Sequence**

11. **Add a Gmail node**
    - Node type: **Gmail**
    - Name it: **Send Personalized Email**
    - Configure Gmail OAuth2 credentials.
    - Send To:
      - `={{ $('HubSpot Contact Created').first().json.properties.email }}`
    - Subject:
      - `={{ $json.personalized_email.subject }}`
    - Message:
      - `={{ $json.personalized_email.body }}`
    - Optional BCC:
      - replace placeholder `user@example.com` with a real internal mailbox if desired
    - Connect the **true** branch of **Score Threshold Check → Send Personalized Email**

12. **Add a Slack node**
    - Node type: **Slack**
    - Name it: **Notify Sales Team**
    - Configure Slack credentials.
    - Choose channel by name:
      - `sales-alerts`
    - Compose a message containing:
      - lead name
      - company
      - title
      - email
      - final score
      - lead category
      - reasoning
    - If you want a HubSpot link, build a real URL using your HubSpot portal and the contact ID. Do not use the raw object ID alone as a URL.
    - Connect the **true** branch of **Score Threshold Check → Notify Sales Team**

13. **Add a Google Sheets node**
    - Node type: **Google Sheets**
    - Name it: **Log Scoring History**
    - Configure Google Sheets credentials.
    - Operation: **Append or Update**
    - Spreadsheet ID: replace `your-lead-scoring-sheet-id` with the actual file ID
    - Sheet name: `Lead Scoring Log`
    - Map columns:
      - Contact Name
      - Email
      - Company
      - Job Title
      - Industry
      - Employee Count
      - Basic Score
      - Final Score
      - Category
      - Reasoning
      - Timestamp
    - Connect **Update Contact Properties → Log Scoring History**

14. **Add sticky notes for documentation**
    - Add one general note with:
      - purpose
      - setup steps
      - customization hints
    - Add section notes for:
      - data enrichment
      - AI-powered scoring
      - lead routing & outreach
      - notifications & tracking

15. **Validate expressions and output structure**
    - Test that HubSpot trigger data actually includes:
      - `properties.email`
      - `properties.firstname`
      - `properties.lastname`
      - `properties.jobtitle`
      - `properties.hs_object_id`
    - Test that Clearbit returns:
      - `company.name`
      - `company.category.industry`
      - `company.metrics.employees`
      - `company.metrics.raised`
    - Test that Gemini output is valid JSON and that `final_score` is numeric.

16. **Harden the workflow before production**
    - Add error handling or fallback logic for:
      - no Clearbit match
      - no AI JSON response
      - HubSpot update failure
      - Gmail/Slack delivery failure
    - Consider adding a parser or structured output validator after **AI Lead Analysis** if model output is inconsistent.

17. **Activate the workflow**
    - Once credentials, mappings, and test data are confirmed, activate the workflow so new HubSpot contacts are processed automatically.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Lead Scoring Automation: This workflow automatically scores and nurtures new HubSpot contacts using Clearbit enrichment and Gemini AI analysis. | Overall workflow purpose |
| How it works: Trigger in HubSpot, enrich with Clearbit, score with rules + AI, route leads, notify sales, log to Google Sheets. | Overall workflow logic |
| Setup steps: Configure HubSpot, Clearbit, Gemini AI credentials; update Slack and Gmail settings; create the Google Sheet; adjust scoring thresholds; customize AI email prompt. | Implementation guidance |
| Customization ideas: Modify target industries, role weights, hot/warm thresholds, email templates, and Slack messages. | Configuration guidance |
| Data enrichment: Clearbit adds company information to score based on size, industry, and funding status. | Enrichment block |
| AI-powered scoring: Gemini AI analyzes the enriched data and generates personalized outreach content. | AI block |
| Lead routing & outreach: Hot leads get personalized emails, warm leads enter nurturing sequences. | Routing block |
| Notifications & tracking: Sales team gets Slack alerts for hot leads. All scores are logged for analysis. | Notification/logging block |

## Implementation Notes

- The workflow has **one entry point**: **HubSpot Contact Created**.
- The workflow has **no sub-workflows** and does not invoke any child workflow.
- The current design defines both `hot_threshold` and `warm_threshold`, but only `hot_threshold` is actually used in routing.
- Several nodes appear intentionally partial and should be completed before production use:
  - **Clearbit Company Enrichment** likely needs query parameters
  - **Update Contact Properties** lacks visible field mappings
  - **Add to Nurturing Sequence** lacks visible field mappings
  - **Log Scoring History** uses a placeholder spreadsheet ID
  - **Notify Sales Team** uses `hs_object_id` as if it were a URL, which should be corrected
- For robustness, add explicit parsing/validation after the AI node so downstream nodes always receive valid JSON.