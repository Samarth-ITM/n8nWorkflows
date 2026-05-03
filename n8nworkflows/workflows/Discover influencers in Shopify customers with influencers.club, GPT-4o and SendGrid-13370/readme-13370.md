Discover influencers in Shopify customers with influencers.club, GPT-4o and SendGrid

https://n8nworkflows.xyz/workflows/discover-influencers-in-shopify-customers-with-influencers-club--gpt-4o-and-sendgrid-13370


# Discover influencers in Shopify customers with influencers.club, GPT-4o and SendGrid

## 1. Workflow Overview

This workflow identifies potential creators or influencers among new Shopify customers, classifies them with AI, generates tailored outreach, then pushes the result into CRM, email, and Slack.

It is designed for e-commerce brands that want to turn existing buyers or new account signups into creator, affiliate, ambassador, or partnership candidates.

### 1.1 Entry Triggers from Shopify
The workflow starts from two separate Shopify webhook triggers:
- a new order event
- a new customer creation event

This allows the brand to capture both:
- buyers with strong purchase intent
- new signups who may still be valuable creators

### 1.2 Email Extraction and Normalization
Each trigger passes through a Set node that reduces the Shopify payload to a simplified structure containing only `email`.

This block is intended to prepare a clean enrichment input, though one path is currently configured incorrectly for Shopify orders.

### 1.3 Creator Enrichment
The workflow calls the influencers.club enrichment node using the extracted email address.

This enrichment attempts to identify whether the customer is a creator and returns social profile data such as platforms, followers, engagement, usernames, bios, and profile URLs.

### 1.4 Routing Based on Creator Detection
An IF node checks whether the enrichment response indicates `result.is_creator === true`.

If true, the workflow continues into AI classification and outreach.  
If false or if enrichment fails, the workflow falls back to a failure-handling branch.

### 1.5 AI Classification
A LangChain agent powered by `gpt-4o-mini` analyzes the enrichment payload and produces a structured creator classification.

It determines:
- primary platform
- tier
- niche and subcategory
- follower count
- engagement
- value score
- audience and brand fit reasoning

### 1.6 AI Outreach Generation
A second LangChain agent powered by `gpt-4o` creates a personalized outreach email in structured JSON.

The outreach adapts to creator tier and context, acknowledging that the person is already a customer.

### 1.7 Operational Outputs
The outreach result fans out to three downstream systems:
- HubSpot contact creation/update
- SendGrid email delivery
- Slack notification for high-value creators

A separate Slack message is also sent for failed enrichment cases.

---

## 2. Block-by-Block Analysis

## 2.1 Workflow Notes and Overview Layer

**Overview:**  
This block contains documentation embedded directly in the canvas through sticky notes. These notes explain the business intent, setup requirements, and the role of each step.

**Nodes Involved:**
- `📋 WORKFLOW OVERVIEW`
- `Sticky Note - Step 1 Triggers`
- `Sticky Note - Step 2`
- `Sticky Note - Step 3 Enrichment`
- `Sticky Note - Step `
- `Sticky Note - Step 5`
- `Sticky Note - Step 6`
- `Sticky Note - Step 7`
- `Sticky Note - Step 8`
- `Sticky Note - Step 9`
- `Sticky Note1`

### Node Details

#### 📋 WORKFLOW OVERVIEW
- **Type and role:** Sticky Note; general workflow summary
- **Configuration choices:** Documents the six main business actions and required credentials
- **Key expressions or variables:** None
- **Input/output connections:** None
- **Version-specific requirements:** Sticky note display only
- **Edge cases or failures:** None
- **Sub-workflow reference:** None

#### Sticky Note - Step 1 Triggers
- **Type and role:** Sticky Note; explains dual Shopify entry points
- **Configuration choices:** Documents trigger use cases and activation behavior
- **Key expressions or variables:** None
- **Input/output connections:** None
- **Version-specific requirements:** None
- **Edge cases or failures:** None
- **Sub-workflow reference:** None

#### Sticky Note - Step 2
- **Type and role:** Sticky Note; explains email preparation
- **Configuration choices:** Notes that the email field should differ by trigger type
- **Key expressions or variables:** Recommends:
  - Order trigger: `{{ $json.customer.email }}`
  - Customer trigger: `{{ $json.email }}`
- **Input/output connections:** None
- **Version-specific requirements:** None
- **Edge cases or failures:** Highlights current risk of wrong email mapping
- **Sub-workflow reference:** None

#### Sticky Note - Step 3 Enrichment
- **Type and role:** Sticky Note; describes enrichment behavior
- **Configuration choices:** Documents expected response structure and API credential requirement
- **Key expressions or variables:** None
- **Input/output connections:** None
- **Version-specific requirements:** None
- **Edge cases or failures:** Mentions API rate limits and failure routing
- **Sub-workflow reference:** None

#### Sticky Note - Step 
- **Type and role:** Sticky Note; documents creator validation logic
- **Configuration choices:** Recommends optional minimum follower threshold
- **Key expressions or variables:** Example:
  - `result.is_creator === true`
  - optional `&& result.max_followers >= 1000`
- **Input/output connections:** None
- **Version-specific requirements:** None
- **Edge cases or failures:** None
- **Sub-workflow reference:** None

#### Sticky Note - Step 5
- **Type and role:** Sticky Note; documents AI classification
- **Configuration choices:** States model cost assumptions, expected JSON output, scoring intent
- **Key expressions or variables:** None
- **Input/output connections:** None
- **Version-specific requirements:** None
- **Edge cases or failures:** None
- **Sub-workflow reference:** None

#### Sticky Note - Step 6
- **Type and role:** Sticky Note; documents outreach generation
- **Configuration choices:** Notes use of GPT-4o, structured output, tier-aware message design
- **Key expressions or variables:** None
- **Input/output connections:** None
- **Version-specific requirements:** None
- **Edge cases or failures:** None
- **Sub-workflow reference:** None

#### Sticky Note - Step 7
- **Type and role:** Sticky Note; documents CRM sync recommendations
- **Configuration choices:** Suggests extra HubSpot property mappings beyond email
- **Key expressions or variables:** Includes recommended expressions such as:
  - `$json.classification.tier`
  - `$json.outreach.program`
- **Input/output connections:** None
- **Version-specific requirements:** None
- **Edge cases or failures:** Warns that custom properties should be created in HubSpot first
- **Sub-workflow reference:** None

#### Sticky Note - Step 8
- **Type and role:** Sticky Note; documents email delivery
- **Configuration choices:** Notes sender identity and generated content use
- **Key expressions or variables:** None
- **Input/output connections:** None
- **Version-specific requirements:** None
- **Edge cases or failures:** None
- **Sub-workflow reference:** None

#### Sticky Note - Step 9
- **Type and role:** Sticky Note; documents Slack alerting
- **Configuration choices:** Shows intended high-value creator alert format
- **Key expressions or variables:** None
- **Input/output connections:** None
- **Version-specific requirements:** None
- **Edge cases or failures:** None
- **Sub-workflow reference:** None

#### Sticky Note1
- **Type and role:** Sticky Note; project description and external link
- **Configuration choices:** Contains a descriptive banner plus link
- **Key expressions or variables:** Link: `https://influencers.club/creatorbook/discover-influencers-among-saas-clients/`
- **Input/output connections:** None
- **Version-specific requirements:** None
- **Edge cases or failures:** None
- **Sub-workflow reference:** None

---

## 2.2 Trigger Block

**Overview:**  
This block listens for two Shopify events and provides two workflow entry points. It broadens capture by including both customers who place orders and customers who only create accounts.

**Nodes Involved:**
- `Shopify Order Trigger`
- `Shopify Customer Trigger`

### Node Details

#### Shopify Order Trigger
- **Type and role:** `n8n-nodes-base.shopifyTrigger`; webhook-based Shopify event listener
- **Configuration choices:**
  - Topic: `orders/create`
  - Authentication: OAuth2
- **Key expressions or variables:** None
- **Input and output connections:**
  - No input
  - Outputs to `Get Order Email`
- **Version-specific requirements:** `typeVersion: 1`
- **Edge cases or potential failures:**
  - Shopify OAuth2 misconfiguration
  - webhook registration failure on activation
  - permissions/scopes insufficient for order events
  - test events may differ from production payloads
- **Sub-workflow reference:** None

#### Shopify Customer Trigger
- **Type and role:** `n8n-nodes-base.shopifyTrigger`; webhook-based Shopify customer creation listener
- **Configuration choices:**
  - Topic: `customers/create`
  - Authentication: OAuth2
- **Key expressions or variables:** None
- **Input and output connections:**
  - No input
  - Outputs to `Get Customer Email`
- **Version-specific requirements:** `typeVersion: 1`
- **Edge cases or potential failures:**
  - OAuth2 issues
  - webhook registration issues
  - missing permissions for customer webhooks
  - payload shape can vary by Shopify version/app scope
- **Sub-workflow reference:** None

---

## 2.3 Data Preparation Block

**Overview:**  
This block normalizes incoming Shopify payloads into a minimal email-only structure for enrichment. It reduces complexity, but one node currently uses the wrong source field for order events.

**Nodes Involved:**
- `Get Order Email`
- `Get Customer Email`

### Node Details

#### Get Order Email
- **Type and role:** `n8n-nodes-base.set`; extracts email for the order path
- **Configuration choices:**
  - Creates a single string field named `email`
  - Current expression: `={{ $json.email }}`
- **Key expressions or variables:**
  - Current: `{{ $json.email }}`
  - Recommended for Shopify orders: `{{ $json.customer.email }}`
- **Input and output connections:**
  - Input from `Shopify Order Trigger`
  - Output to `Enrich by Email`
- **Version-specific requirements:** `typeVersion: 3.4`
- **Edge cases or potential failures:**
  - order payload may not expose the buyer email at root level
  - if `email` is empty, enrichment may fail or waste API calls
  - guest checkout and unusual order payloads may require fallback logic
- **Sub-workflow reference:** None

#### Get Customer Email
- **Type and role:** `n8n-nodes-base.set`; extracts email for the customer path
- **Configuration choices:**
  - Creates a single string field named `email`
  - Expression: `={{ $json.email }}`
- **Key expressions or variables:**
  - `{{ $json.email }}`
- **Input and output connections:**
  - Input from `Shopify Customer Trigger`
  - Output to `Enrich by Email`
- **Version-specific requirements:** `typeVersion: 3.4`
- **Edge cases or potential failures:**
  - customer webhook without email would break downstream enrichment
  - no validation is present for malformed email addresses
- **Sub-workflow reference:** None

---

## 2.4 Enrichment and Failure Branch Block

**Overview:**  
This block performs creator discovery by email using influencers.club. It is configured to continue on error and expose a fallback path for failures or non-creator cases.

**Nodes Involved:**
- `Enrich by Email`
- `Handle Failed Enrichment`
- `Send a message - Failed Enrichment`

### Node Details

#### Enrich by Email
- **Type and role:** `n8n-nodes-influencersclub.influencersClub`; creator enrichment lookup by email
- **Configuration choices:**
  - Email parameter: `={{ $json.email }}`
  - `onError: continueErrorOutput`
  - Uses influencers.club API credentials
- **Key expressions or variables:**
  - `{{ $json.email }}`
- **Input and output connections:**
  - Inputs from `Get Order Email` and `Get Customer Email`
  - Main output index 0 to `Is Creator?`
  - Error output index 1 to `Handle Failed Enrichment`
- **Version-specific requirements:** `typeVersion: 1`
- **Edge cases or potential failures:**
  - missing/invalid API key
  - rate limiting
  - email not found
  - malformed response
  - if email is blank, request may fail or return no creator
- **Sub-workflow reference:** None

#### Handle Failed Enrichment
- **Type and role:** `n8n-nodes-base.code`; builds a normalized fallback payload for failed enrichment
- **Configuration choices:**
  - Reads first input item
  - Returns a fixed JSON structure marking:
    - `enrichment_status: 'failed'`
    - `creator_status: 'non_creator'`
    - routing metadata for standard newsletter flow
    - current ISO timestamp
- **Key expressions or variables:**
  - JavaScript uses `$input.first().json`
  - `new Date().toISOString()`
- **Input and output connections:**
  - Input from `Enrich by Email` error output
  - Output to `Send a message - Failed Enrichment`
- **Version-specific requirements:** `typeVersion: 2`
- **Edge cases or potential failures:**
  - if incoming payload lacks `email`, `first_name`, or `last_name`, returned object fields may be undefined
  - code node runtime errors if input is unexpectedly empty
- **Sub-workflow reference:** None

#### Send a message - Failed Enrichment
- **Type and role:** `n8n-nodes-base.slack`; operational alert for failed enrichment cases
- **Configuration choices:**
  - OAuth2 authentication
  - Sends to Slack channel `social-listening`
  - Text: `Check the Newsletter Subscriber flow for {{$json.email}}`
- **Key expressions or variables:**
  - `{{ $json.email }}`
- **Input and output connections:**
  - Input from `Handle Failed Enrichment`
  - No downstream output
- **Version-specific requirements:** `typeVersion: 2.4`
- **Edge cases or potential failures:**
  - Slack OAuth token invalid
  - channel access missing
  - undefined email leads to a vague alert
- **Sub-workflow reference:** None

---

## 2.5 Creator Validation Block

**Overview:**  
This block prevents unnecessary AI processing by allowing only enriched creator records to continue. It acts as the cost-control gate before classification and outreach.

**Nodes Involved:**
- `Is Creator?`

### Node Details

#### Is Creator?
- **Type and role:** `n8n-nodes-base.if`; conditional route based on enrichment result
- **Configuration choices:**
  - Condition checks boolean true on `{{ $json.result.is_creator }}`
  - Strict type validation enabled
  - Note attached: "Check if the enrichment returned is_creator = true"
- **Key expressions or variables:**
  - `{{ $json.result.is_creator }}`
- **Input and output connections:**
  - Input from `Enrich by Email`
  - True output to `AI Creator Classification Agent`
  - False output is not connected
- **Version-specific requirements:** `typeVersion: 2.2`
- **Edge cases or potential failures:**
  - if `result.is_creator` is missing or not strictly boolean, condition may evaluate false
  - false branch currently ends silently; non-creators are not logged unless the enrichment node actually errors
- **Sub-workflow reference:** None

---

## 2.6 AI Classification Block

**Overview:**  
This block uses a LangChain agent with an OpenAI chat model and structured output parser to convert raw enrichment data into a standardized creator classification.

**Nodes Involved:**
- `AI Creator Classification Agent`
- `OpenAI Classifier Model1`
- `Classification Output Parser1`

### Node Details

#### AI Creator Classification Agent
- **Type and role:** `@n8n/n8n-nodes-langchain.agent`; orchestrates LLM-based creator classification
- **Configuration choices:**
  - Prompt type: defined in node
  - Input text: `={{ JSON.stringify($json) }}`
  - Has output parser enabled
  - Large, explicit system prompt describing:
    - tier classification rules
    - niche taxonomy
    - value score
    - brand fit
    - content themes
    - audience demographics
    - reasoning format
- **Key expressions or variables:**
  - `{{ JSON.stringify($json) }}`
- **Input and output connections:**
  - Main input from `Is Creator?`
  - AI language model input from `OpenAI Classifier Model1`
  - AI output parser input from `Classification Output Parser1`
  - Main output to `E-Commerce Creator Outreach Agent`
- **Version-specific requirements:** `typeVersion: 3.1`; requires LangChain-compatible n8n version
- **Edge cases or potential failures:**
  - model may return schema-invalid JSON
  - prompt assumes platform arrays and bio details exist
  - sparse enrichment data may reduce classification quality
  - token usage/cost issues for large payloads
- **Sub-workflow reference:** None

#### OpenAI Classifier Model1
- **Type and role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`; LLM backend for classification
- **Configuration choices:**
  - Model: `gpt-4o-mini`
  - Temperature: `0.3`
- **Key expressions or variables:** None
- **Input and output connections:**
  - AI language model output to `AI Creator Classification Agent`
- **Version-specific requirements:** `typeVersion: 1.3`
- **Edge cases or potential failures:**
  - OpenAI auth errors
  - unavailable model in account/region
  - quota exhaustion
  - timeout or rate limit
- **Sub-workflow reference:** None

#### Classification Output Parser1
- **Type and role:** `@n8n/n8n-nodes-langchain.outputParserStructured`; enforces JSON schema for classification output
- **Configuration choices:**
  - Manual JSON schema
  - Requires root object `classification`
  - Defines enums and numeric constraints for:
    - `primary_platform`
    - `tier`
    - `niche`
    - `value_score`
    - `brand_fit_score`
    - and other descriptive fields
- **Key expressions or variables:** None
- **Input and output connections:**
  - AI output parser output to `AI Creator Classification Agent`
- **Version-specific requirements:** `typeVersion: 1.2`
- **Edge cases or potential failures:**
  - invalid model output causing parser rejection
  - schema too strict for unexpected but plausible platform values
  - missing required keys such as `reasoning`
- **Sub-workflow reference:** None

---

## 2.7 AI Outreach Generation Block

**Overview:**  
This block generates a structured outreach message tailored to the creator’s tier, niche, and existing customer relationship. It uses a second LLM and parser to produce clean JSON for downstream systems.

**Nodes Involved:**
- `E-Commerce Creator Outreach Agent`
- `OpenAI Chat Model1`
- `Structured Output Parser1`

### Node Details

#### E-Commerce Creator Outreach Agent
- **Type and role:** `@n8n/n8n-nodes-langchain.agent`; generates outreach subject/body and program recommendation
- **Configuration choices:**
  - Input text: `={{ JSON.stringify($json) }}`
  - Has output parser enabled
  - Detailed system prompt with hard rules for:
    - acknowledging customer status
    - tier-based messaging
    - message length
    - subject line rules
    - offer matching
    - talking points
- **Key expressions or variables:**
  - `{{ JSON.stringify($json) }}`
- **Input and output connections:**
  - Main input from `AI Creator Classification Agent`
  - AI language model input from `OpenAI Chat Model1`
  - AI output parser input from `Structured Output Parser1`
  - Main outputs to:
    - `Send a message`
    - `Create or update a CRM contact`
    - `Send an email`
- **Version-specific requirements:** `typeVersion: 3.1`
- **Edge cases or potential failures:**
  - parser mismatch if model output does not conform
  - prompt assumes fields like first name and order context may exist, but current upstream mapping may not preserve them
  - generated body may contain formatting unsuitable for plain-text email depending on SendGrid expectations
- **Sub-workflow reference:** None

#### OpenAI Chat Model1
- **Type and role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`; LLM backend for outreach generation
- **Configuration choices:**
  - Model: `gpt-4o`
  - Temperature: `0.8`
- **Key expressions or variables:** None
- **Input and output connections:**
  - AI language model output to `E-Commerce Creator Outreach Agent`
- **Version-specific requirements:** `typeVersion: 1.3`
- **Edge cases or potential failures:**
  - auth issues
  - cost increase relative to classifier model
  - rate limits and timeouts
- **Sub-workflow reference:** None

#### Structured Output Parser1
- **Type and role:** `@n8n/n8n-nodes-langchain.outputParserStructured`; validates outreach JSON
- **Configuration choices:**
  - Manual JSON schema
  - Requires root `outreach` object
  - Required fields:
    - `subject`
    - `message`
    - `priority`
    - `program`
    - `offer_type`
    - `talking_points`
- **Key expressions or variables:** None
- **Input and output connections:**
  - AI output parser output to `E-Commerce Creator Outreach Agent`
- **Version-specific requirements:** `typeVersion: 1.2`
- **Edge cases or potential failures:**
  - invalid enum values from model
  - missing fields
  - non-array `talking_points`
- **Sub-workflow reference:** None

---

## 2.8 CRM, Email, and Slack Delivery Block

**Overview:**  
This block operationalizes the AI output. It sends the outreach email, updates HubSpot, and posts a Slack alert.

**Nodes Involved:**
- `Create or update a CRM contact`
- `Send an email`
- `Send a message`

### Node Details

#### Create or update a CRM contact
- **Type and role:** `n8n-nodes-base.hubspot`; creates or updates a contact in HubSpot
- **Configuration choices:**
  - Authentication: app token
  - Uses email field from enrichment node:
    - `={{ $('Influencers.club - Enrichment API by Email').first().json.result.email }}`
  - No additional fields mapped
- **Key expressions or variables:**
  - `$('Influencers.club - Enrichment API by Email').first().json.result.email`
- **Input and output connections:**
  - Input from `E-Commerce Creator Outreach Agent`
  - No downstream output
- **Version-specific requirements:** `typeVersion: 2.2`
- **Edge cases or potential failures:**
  - **Important:** the referenced node name does not exist in this workflow
  - actual enrichment node is named `Enrich by Email`
  - this expression will fail unless corrected
  - missing HubSpot scopes or invalid app token
  - if contact properties are later added, they must already exist in HubSpot
- **Sub-workflow reference:** None

#### Send an email
- **Type and role:** `n8n-nodes-base.sendGrid`; sends AI-generated outreach email
- **Configuration choices:**
  - Resource: mail
  - Subject: `={{ $json.output.outreach.subject }}`
  - To: `={{ $('Influencers.club - Enrichment API by Email').item.json.result.email }}`
  - From name: `Gjorgji P`
  - From email: `user@example.com`
  - Content: `={{ $json.output.outreach.message }}`
- **Key expressions or variables:**
  - `$json.output.outreach.subject`
  - `$json.output.outreach.message`
  - `$('Influencers.club - Enrichment API by Email').item.json.result.email`
- **Input and output connections:**
  - Input from `E-Commerce Creator Outreach Agent`
  - No downstream output
- **Version-specific requirements:** `typeVersion: 1`
- **Edge cases or potential failures:**
  - **Important:** references a non-existent node name
  - sender address may be unverified in SendGrid
  - generated output path depends on agent output being under `output.outreach`
  - if the agent returns a different top-level structure, expressions fail
- **Sub-workflow reference:** None

#### Send a message
- **Type and role:** `n8n-nodes-base.slack`; notifies Slack about a discovered creator
- **Configuration choices:**
  - OAuth2 authentication
  - Sends to channel `social-listening`
  - Message includes name, email, tier, and platform
- **Key expressions or variables:**
  - `{{ $('Influencers.club - Enrichment API by Email').item.json.result.tiktok.full_name }}`
  - `{{ $('Influencers.club - Enrichment API by Email').item.json.result.email }}`
  - `{{ $('AI Creator Classification Agent').item.json.output.classification.tier }}`
  - `{{ $('AI Creator Classification Agent').item.json.output.classification.primary_platform }}`
- **Input and output connections:**
  - Input from `E-Commerce Creator Outreach Agent`
  - No downstream output
- **Version-specific requirements:** `typeVersion: 2.4`
- **Edge cases or potential failures:**
  - **Important:** again references a non-existent enrichment node name
  - assumes a `result.tiktok.full_name` structure, which may not exist if the primary platform is not TikTok or the enrichment schema differs
  - assumes classifier output is under `output.classification`
  - Slack token/channel access issues
- **Sub-workflow reference:** None

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| 📋 WORKFLOW OVERVIEW | Sticky Note | Embedded workflow overview and setup requirements |  |  | ## 🎯 WORKFLOW OVERVIEW<br>**How it works**<br>1. Captures Shopify orders/customers<br>2. Enriches emails with social media data<br>3. Classifies creators by tier & niche<br>4. Sends personalized partnership outreach<br>5. Syncs to HubSpot CRM<br>6. Alerts team on Slack<br>**Set up required**<br>- Shopify app key (trigger)<br>- HubSpot App Token (read + write)<br>- influencers.club API key<br>- OpenAI API key<br>- Slack API key<br>- Sendgrid API key |
| Sticky Note - Step 1 Triggers | Sticky Note | Documentation for Shopify trigger strategy |  |  | ## 🔔 STEP 1: TRIGGERS<br>**Two entry points:**<br>**A) Shopify Order Trigger**<br>- Fires when: New order created<br>- Use case: Convert buyers into affiliates<br>- Best for: Product purchases<br>**B) Shopify Customer Trigger**<br>- Fires when: New customer account created<br>- Use case: Newsletter signups, account creation<br>- Best for: Early funnel engagement<br>**Setup:**<br>1. Connect Shopify OAuth2 account<br>2. Webhooks auto-register on activation<br>3. Test with Shopify test orders<br>**Why two triggers?**<br>Captures both:<br>- Paying customers (higher intent)<br>- Free signups (broader reach) |
| Sticky Note - Step 2 | Sticky Note | Documentation for email extraction |  |  | ## 🔧 STEP 2: DATA PREPARATION<br>**Edit Fields / Get Customer Email**<br>**Purpose:** Extract clean email address from Shopify payload<br>**Why needed:**<br>- Shopify webhooks contain 50+ fields<br>- Enrichment API only needs email<br>- Simplifies downstream processing<br>**⚠️ TODO - Replace empty email field with:**<br>`{{ $json.customer.email }}` for order trigger<br>`{{ $json.email }}` for customer trigger<br>**Best practice:** Validate email format before API call to avoid wasted credits |
| Shopify Order Trigger | n8n-nodes-base.shopifyTrigger | Entry point for new Shopify orders |  | Get Order Email | ## 🔔 STEP 1: TRIGGERS<br>**Two entry points:**<br>**A) Shopify Order Trigger**<br>- Fires when: New order created<br>- Use case: Convert buyers into affiliates<br>- Best for: Product purchases<br>**B) Shopify Customer Trigger**<br>- Fires when: New customer account created<br>- Use case: Newsletter signups, account creation<br>- Best for: Early funnel engagement<br>**Setup:**<br>1. Connect Shopify OAuth2 account<br>2. Webhooks auto-register on activation<br>3. Test with Shopify test orders<br>**Why two triggers?**<br>Captures both:<br>- Paying customers (higher intent)<br>- Free signups (broader reach) |
| Shopify Customer Trigger | n8n-nodes-base.shopifyTrigger | Entry point for new Shopify customers |  | Get Customer Email | ## 🔔 STEP 1: TRIGGERS<br>**Two entry points:**<br>**A) Shopify Order Trigger**<br>- Fires when: New order created<br>- Use case: Convert buyers into affiliates<br>- Best for: Product purchases<br>**B) Shopify Customer Trigger**<br>- Fires when: New customer account created<br>- Use case: Newsletter signups, account creation<br>- Best for: Early funnel engagement<br>**Setup:**<br>1. Connect Shopify OAuth2 account<br>2. Webhooks auto-register on activation<br>3. Test with Shopify test orders<br>**Why two triggers?**<br>Captures both:<br>- Paying customers (higher intent)<br>- Free signups (broader reach) |
| Get Order Email | n8n-nodes-base.set | Extracts email from order payload | Shopify Order Trigger | Enrich by Email | ## 🔧 STEP 2: DATA PREPARATION<br>**Edit Fields / Get Customer Email**<br>**Purpose:** Extract clean email address from Shopify payload<br>**Why needed:**<br>- Shopify webhooks contain 50+ fields<br>- Enrichment API only needs email<br>- Simplifies downstream processing<br>**⚠️ TODO - Replace empty email field with:**<br>`{{ $json.customer.email }}` for order trigger<br>`{{ $json.email }}` for customer trigger<br>**Best practice:** Validate email format before API call to avoid wasted credits |
| Get Customer Email | n8n-nodes-base.set | Extracts email from customer payload | Shopify Customer Trigger | Enrich by Email | ## 🔧 STEP 2: DATA PREPARATION<br>**Edit Fields / Get Customer Email**<br>**Purpose:** Extract clean email address from Shopify payload<br>**Why needed:**<br>- Shopify webhooks contain 50+ fields<br>- Enrichment API only needs email<br>- Simplifies downstream processing<br>**⚠️ TODO - Replace empty email field with:**<br>`{{ $json.customer.email }}` for order trigger<br>`{{ $json.email }}` for customer trigger<br>**Best practice:** Validate email format before API call to avoid wasted credits |
| Sticky Note - Step 3 Enrichment | Sticky Note | Documentation for creator enrichment |  |  | ## 🔍 STEP 3: CREATOR ENRICHMENT<br>**Influencers.club API Call**<br>**What it does:**<br>- Searches 50+ social platforms for email<br>- Returns creator profiles if found<br>- Includes: followers, engagement, bio, URLs<br>**Response structure:** includes `result.is_creator`, `email`, `platforms`<br>**Error handling:**<br>- On error → Continue to "Handle Failed Enrichment"<br>- Non-creators → is_creator = false<br>- Rate limits: 100/min, 10K/day<br>**⚙️ Setup required:** Add API key in HTTP Header Auth credentials |
| Enrich by Email | n8n-nodes-influencersclub.influencersClub | Enriches email with creator/social profile data | Get Order Email; Get Customer Email | Is Creator?; Handle Failed Enrichment | ## 🔍 STEP 3: CREATOR ENRICHMENT<br>**Influencers.club API Call**<br>**What it does:**<br>- Searches 50+ social platforms for email<br>- Returns creator profiles if found<br>- Includes: followers, engagement, bio, URLs<br>**Response structure:** includes `result.is_creator`, `email`, `platforms`<br>**Error handling:**<br>- On error → Continue to "Handle Failed Enrichment"<br>- Non-creators → is_creator = false<br>- Rate limits: 100/min, 10K/day<br>**⚙️ Setup required:** Add API key in HTTP Header Auth credentials |
| Sticky Note - Step  | Sticky Note | Documentation for creator validation |  |  | ## ✅ STEP 4: CREATOR VALIDATION<br>**Is Creator? (IF Node)**<br>Logic routes true creators to classification and skips others.<br>Optimization tip: add minimum follower threshold such as `result.is_creator === true && result.max_followers >= 1000` |
| Is Creator? | n8n-nodes-base.if | Filters enriched records to creators only | Enrich by Email | AI Creator Classification Agent | ## ✅ STEP 4: CREATOR VALIDATION<br>**Is Creator? (IF Node)**<br>Logic routes true creators to classification and skips others.<br>Optimization tip: add minimum follower threshold such as `result.is_creator === true && result.max_followers >= 1000` |
| Sticky Note - Step 5 | Sticky Note | Documentation for AI classification |  |  | ## 🏷️ STEP 5: AI CLASSIFICATION<br>**AI Creator Classification Agent**<br>Analyzes enrichment data, assigns tier and niche, and outputs structured JSON using GPT-4o-mini |
| OpenAI Classifier Model1 | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM backend for creator classification |  | AI Creator Classification Agent | ## 🏷️ STEP 5: AI CLASSIFICATION<br>**AI Creator Classification Agent**<br>Analyzes enrichment data, assigns tier and niche, and outputs structured JSON using GPT-4o-mini |
| Classification Output Parser1 | @n8n/n8n-nodes-langchain.outputParserStructured | Structured parser for classification JSON |  | AI Creator Classification Agent | ## 🏷️ STEP 5: AI CLASSIFICATION<br>**AI Creator Classification Agent**<br>Analyzes enrichment data, assigns tier and niche, and outputs structured JSON using GPT-4o-mini |
| AI Creator Classification Agent | @n8n/n8n-nodes-langchain.agent | Produces normalized creator classification | Is Creator?; OpenAI Classifier Model1; Classification Output Parser1 | E-Commerce Creator Outreach Agent | ## 🏷️ STEP 5: AI CLASSIFICATION<br>**AI Creator Classification Agent**<br>Analyzes enrichment data, assigns tier and niche, and outputs structured JSON using GPT-4o-mini |
| Sticky Note - Step 6 | Sticky Note | Documentation for outreach generation |  |  | ## ✉️ STEP 6: AI OUTREACH GENERATION<br>**E-Commerce Creator Outreach Agent**<br>Generates personalized email subject and body using GPT-4o with structured JSON output |
| OpenAI Chat Model1 | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM backend for outreach generation |  | E-Commerce Creator Outreach Agent | ## ✉️ STEP 6: AI OUTREACH GENERATION<br>**E-Commerce Creator Outreach Agent**<br>Generates personalized email subject and body using GPT-4o with structured JSON output |
| Structured Output Parser1 | @n8n/n8n-nodes-langchain.outputParserStructured | Structured parser for outreach JSON |  | E-Commerce Creator Outreach Agent | ## ✉️ STEP 6: AI OUTREACH GENERATION<br>**E-Commerce Creator Outreach Agent**<br>Generates personalized email subject and body using GPT-4o with structured JSON output |
| E-Commerce Creator Outreach Agent | @n8n/n8n-nodes-langchain.agent | Generates creator outreach payload | AI Creator Classification Agent; OpenAI Chat Model1; Structured Output Parser1 | Send a message; Create or update a CRM contact; Send an email | ## ✉️ STEP 6: AI OUTREACH GENERATION<br>**E-Commerce Creator Outreach Agent**<br>Generates personalized email subject and body using GPT-4o with structured JSON output |
| Sticky Note - Step 7 | Sticky Note | Documentation for HubSpot sync |  |  | ## 💾 STEP 7: CRM SYNC<br>**HubSpot Contact Update**<br>Creates or updates HubSpot contacts and recommends adding creator-specific properties |
| Create or update a CRM contact | n8n-nodes-base.hubspot | Upserts contact in HubSpot | E-Commerce Creator Outreach Agent |  | ## 💾 STEP 7: CRM SYNC<br>**HubSpot Contact Update**<br>Creates or updates HubSpot contacts and recommends adding creator-specific properties |
| Sticky Note - Step 8 | Sticky Note | Documentation for SendGrid delivery |  |  | ## 📧 STEP 8: EMAIL DELIVERY<br>**SendGrid Email**<br>Sends AI-generated outreach email from a partnership sender address |
| Send an email | n8n-nodes-base.sendGrid | Sends outreach email to creator | E-Commerce Creator Outreach Agent |  | ## 📧 STEP 8: EMAIL DELIVERY<br>**SendGrid Email**<br>Sends AI-generated outreach email from a partnership sender address |
| Sticky Note - Step 9 | Sticky Note | Documentation for Slack creator alerts |  |  | ## 🔔 STEP 9: SLACK NOTIFICATIONS<br>**High-Value Creator Alerts**<br>Notifies Slack with creator name, email, tier, platform, priority, and program |
| Send a message | n8n-nodes-base.slack | Sends Slack alert for creator discovery | E-Commerce Creator Outreach Agent |  | ## 🔔 STEP 9: SLACK NOTIFICATIONS<br>**High-Value Creator Alerts**<br>Notifies Slack with creator name, email, tier, platform, priority, and program |
| Handle Failed Enrichment | n8n-nodes-base.code | Creates fallback payload for enrichment errors | Enrich by Email | Send a message - Failed Enrichment |  |
| Send a message - Failed Enrichment | n8n-nodes-base.slack | Sends Slack alert for enrichment failures | Handle Failed Enrichment |  |  |
| Sticky Note1 | Sticky Note | Banner note with external explanation link |  |  | ## Find and activate creators inside your loyalty program – API Creatorbooks<br>Step by step workflow to enrich ecommerce customers emails on Shopify with multi social (Instagram, Tiktok, Youtube, Twitter, Onlyfans, Twitch and more) and launch personalized comms using the influencer.club API. [Full explanation](https://influencers.club/creatorbook/discover-influencers-among-saas-clients/) |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and give it a name similar to:  
   `Discover influencers by analyzing Shopify customers and executing tailored outreach`.

2. **Add a Shopify Trigger node** named `Shopify Order Trigger`.
   - Type: `Shopify Trigger`
   - Event/topic: `orders/create`
   - Authentication: `OAuth2`
   - Connect Shopify OAuth2 credentials with webhook permissions for order events.

3. **Add a second Shopify Trigger node** named `Shopify Customer Trigger`.
   - Type: `Shopify Trigger`
   - Event/topic: `customers/create`
   - Authentication: `OAuth2`
   - Use the same Shopify credential if appropriate.

4. **Add a Set node** named `Get Order Email`.
   - Add one field:
     - `email` as string
   - Recommended expression:
     - `{{ $json.customer.email }}`
   - This is the correct mapping for the order path.
   - Connect `Shopify Order Trigger` → `Get Order Email`.

5. **Add another Set node** named `Get Customer Email`.
   - Add one field:
     - `email` as string
   - Expression:
     - `{{ $json.email }}`
   - Connect `Shopify Customer Trigger` → `Get Customer Email`.

6. **Add the influencers.club node** named `Enrich by Email`.
   - Type: `Influencers Club`
   - Operation: enrich/search by email
   - Email field:
     - `{{ $json.email }}`
   - Set the node to continue on error / use error output if available.
   - Configure influencers.club API credentials.
   - Connect both:
     - `Get Order Email` → `Enrich by Email`
     - `Get Customer Email` → `Enrich by Email`

7. **Add an IF node** named `Is Creator?`.
   - Condition:
     - left value: `{{ $json.result.is_creator }}`
     - operator: `is true`
   - Use strict type validation if available.
   - Connect `Enrich by Email` main success output → `Is Creator?`

8. **Add a Code node** named `Handle Failed Enrichment`.
   - Use it for the enrichment error output.
   - Paste logic equivalent to:
     - copy incoming email/name data
     - mark `enrichment_status` as `failed`
     - set creator classification to non-creator defaults
     - add a timestamp
   - Connect `Enrich by Email` error output → `Handle Failed Enrichment`.

9. **Add a Slack node** named `Send a message - Failed Enrichment`.
   - Operation: send message to channel
   - Authentication: Slack OAuth2
   - Channel: your monitoring channel, e.g. `#social-listening`
   - Text:
     - `Check the Newsletter Subscriber flow for {{$json.email}}`
   - Connect `Handle Failed Enrichment` → `Send a message - Failed Enrichment`.

10. **Add an OpenAI Chat Model node** named `OpenAI Classifier Model1`.
    - Type: LangChain OpenAI Chat Model
    - Model: `gpt-4o-mini`
    - Temperature: `0.3`
    - Configure OpenAI API credentials.

11. **Add a Structured Output Parser node** named `Classification Output Parser1`.
    - Schema type: manual
    - Define a schema with root key `classification`
    - Include fields such as:
      - `primary_platform`
      - `tier`
      - `tier_name`
      - `niche`
      - `niche_name`
      - `niche_subcategory`
      - `followers`
      - `engagement_rate`
      - `username`
      - `profile_url`
      - `value_score`
      - `content_themes`
      - `audience_demographics`
      - `brand_fit_score`
      - `reasoning`

12. **Add an AI Agent node** named `AI Creator Classification Agent`.
    - Type: LangChain Agent
    - Prompt mode: define
    - Input text:
      - `{{ JSON.stringify($json) }}`
    - Add the long-form system instruction that:
      - selects a primary platform
      - classifies tier from follower count
      - detects niche/subcategory
      - calculates value score and brand fit
      - returns valid JSON only
    - Enable output parser.
    - Connect:
      - `Is Creator?` true output → `AI Creator Classification Agent`
      - `OpenAI Classifier Model1` → AI language model port
      - `Classification Output Parser1` → AI output parser port

13. **Add a second OpenAI Chat Model node** named `OpenAI Chat Model1`.
    - Model: `gpt-4o`
    - Temperature: `0.8`
    - Reuse OpenAI credentials.

14. **Add another Structured Output Parser node** named `Structured Output Parser1`.
    - Schema type: manual
    - Root key: `outreach`
    - Required fields:
      - `subject`
      - `message`
      - `priority`
      - `program`
      - `offer_type`
      - `talking_points`

15. **Add another AI Agent node** named `E-Commerce Creator Outreach Agent`.
    - Input text:
      - `{{ JSON.stringify($json) }}`
    - Use a system prompt that:
      - acknowledges the person is already a customer
      - adapts tone and offer based on creator tier
      - outputs JSON only
      - includes subject, message, priority, program, offer type, and talking points
    - Enable output parser.
    - Connect:
      - `AI Creator Classification Agent` → `E-Commerce Creator Outreach Agent`
      - `OpenAI Chat Model1` → AI language model port
      - `Structured Output Parser1` → AI output parser port

16. **Add a HubSpot node** named `Create or update a CRM contact`.
    - Type: HubSpot
    - Operation: create or update contact
    - Authentication: App Token
    - Use email as unique identifier.
    - Recommended expression:
      - `{{ $('Enrich by Email').first().json.result.email }}`
    - Configure HubSpot app token credentials with contact read/write scopes.
    - Optional: add mapped custom properties such as tier, niche, followers, value score, program, and priority.

17. **Add a SendGrid node** named `Send an email`.
    - Resource: mail
    - To email:
      - `{{ $('Enrich by Email').item.json.result.email }}`
    - Subject:
      - If your agent output is under `output.outreach.subject`, use that exact path.
      - Otherwise adapt to the actual execution data, e.g. `{{ $json.outreach.subject }}`
    - Content/body:
      - same rule as above, e.g. `{{ $json.output.outreach.message }}` or `{{ $json.outreach.message }}`
    - From name: your partnerships team name
    - From email: a verified sender such as `partnerships@yourdomain.com`
    - Configure SendGrid API credentials.

18. **Add a Slack node** named `Send a message`.
    - Operation: send channel message
    - Channel: `#social-listening` or equivalent
    - Build the message from creator data, for example:
      - name from enrichment or classification
      - email from enrichment
      - tier and primary platform from classification
    - Prefer expressions referencing actual existing nodes, e.g.:
      - `{{ $('Enrich by Email').item.json.result.email }}`
      - `{{ $('AI Creator Classification Agent').item.json.output.classification.tier }}`
    - Configure Slack OAuth2 credentials.

19. **Connect the final fan-out**:
    - `E-Commerce Creator Outreach Agent` → `Create or update a CRM contact`
    - `E-Commerce Creator Outreach Agent` → `Send an email`
    - `E-Commerce Creator Outreach Agent` → `Send a message`

20. **Correct broken node references before activating.**
    - In the provided workflow JSON, several expressions reference a non-existent node named:
      - `Influencers.club - Enrichment API by Email`
    - Replace those references with the actual node name:
      - `Enrich by Email`

21. **Validate the actual output shape of the AI Agent nodes.**
    - Depending on your n8n/LangChain version, parsed data may appear under:
      - `$json.output.classification`
      - `$json.classification`
      - `$json.output.outreach`
      - `$json.outreach`
    - Run the workflow manually once and update downstream expressions accordingly.

22. **Optionally improve the false branch of `Is Creator?`.**
    - Add a node to log or notify on non-creator results.
    - Currently only hard failures are handled; false evaluations just stop.

23. **Activate the workflow.**
    - Ensure Shopify webhooks register successfully.
    - Test with:
      - a Shopify test order
      - a customer creation event
      - a known creator email
      - a non-creator email
      - a malformed or missing email case

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Find and activate creators inside your loyalty program – API Creatorbooks | Canvas note/banner |
| Step by step workflow to enrich ecommerce customers emails on Shopify with multi social (Instagram, Tiktok, Youtube, Twitter, Onlyfans, Twitch and more) and launch personalized comms using the influencer.club API | https://influencers.club/creatorbook/discover-influencers-among-saas-clients/ |
| Main implementation risk: several downstream expressions refer to a node name that does not exist in this workflow | Replace `Influencers.club - Enrichment API by Email` with `Enrich by Email` |
| Main data-mapping risk: the order email extraction is likely wrong | Use `{{ $json.customer.email }}` in `Get Order Email` |
| Main routing gap: non-creators detected successfully by enrichment are not explicitly handled | Add a false-branch node after `Is Creator?` if you want logging or alternate nurturing |