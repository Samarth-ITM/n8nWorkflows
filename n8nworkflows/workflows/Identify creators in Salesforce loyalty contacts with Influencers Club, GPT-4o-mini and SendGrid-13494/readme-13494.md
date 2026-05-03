Identify creators in Salesforce loyalty contacts with Influencers Club, GPT-4o-mini and SendGrid

https://n8nworkflows.xyz/workflows/identify-creators-in-salesforce-loyalty-contacts-with-influencers-club--gpt-4o-mini-and-sendgrid-13494


# Identify creators in Salesforce loyalty contacts with Influencers Club, GPT-4o-mini and SendGrid

# 1. Workflow Overview

This workflow identifies potential creators or influencers among newly created Salesforce loyalty contacts, enriches their social profile using Influencers Club, classifies them with GPT-4o-mini, decides the best outreach and ambassador activation strategy, updates Salesforce with creator-related metadata, and can send a personalized email via SendGrid.

Its main use case is creator discovery inside an existing customer or loyalty database, so brands can detect high-potential creators already in their CRM and route them into ambassador, reward, or nurture programs.

## 1.1 Trigger and Contact Intake
The workflow starts when a new Salesforce Contact is created. It extracts a compact set of fields needed for downstream enrichment and personalization, such as email, name, loyalty tier, and customer value data.

## 1.2 Creator Enrichment
The extracted email is sent to Influencers Club to retrieve creator/social profile information across supported platforms.

## 1.3 AI Classification
The enriched creator profile is analyzed by a LangChain AI agent using OpenAI GPT-4o-mini. The model returns a structured creator classification including primary platform, tier, niche, value score, engagement, audience, and reasoning.

## 1.4 Routing and Personalization
A custom code node converts classification results into a concrete outreach strategy. In parallel, a second AI agent generates personalized messaging content aligned with the routing context.

## 1.5 Ambassador Activation Decision
The routing data and the personalization result are merged, then a second code node assigns a formal ambassador activation level such as Elite, Core, Rising Star, or none.

## 1.6 CRM Update and Optional Email Dispatch
The workflow routes all activation outcomes through a Switch node into a Salesforce Contact update. It writes creator and activation metadata back into custom Salesforce fields. A final SendGrid node exists for personalized email delivery, but it is currently disabled.

---

# 2. Block-by-Block Analysis

## Block 1: Trigger and Contact Field Extraction

### Overview
This block detects the creation of a new Salesforce contact and normalizes the contact data into a smaller object for the rest of the workflow. It ensures the enrichment stage receives the email and customer context in a predictable structure.

### Nodes Involved
- Salesforce Trigger
- Extract Data Fields
- Sticky Note - Header
- Sticky Note
- Sticky Note12

### Node Details

#### Salesforce Trigger
- **Type and technical role:** `n8n-nodes-base.salesforceTrigger`; polling trigger for new Salesforce contacts.
- **Configuration choices:**  
  - Trigger condition: `contactCreated`
  - Polling frequency: every minute
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  - Entry point node
  - Outputs to `Extract Data Fields`
- **Version-specific requirements:** Uses node version 1.
- **Edge cases or potential failure types:**  
  - Salesforce credential or OAuth failure
  - Insufficient API permissions
  - Polling lag or missed events depending on Salesforce API behavior
  - If the Salesforce Contact object does not expose expected fields, downstream expressions may resolve to empty values
- **Sub-workflow reference:** None.

#### Extract Data Fields
- **Type and technical role:** `n8n-nodes-base.set`; creates a curated payload from Salesforce contact data.
- **Configuration choices:**  
  - Maps the following fields:
    - `email` ← `{{$json.Email}}`
    - `member_name` ← `{{$json.FirstName}} {{$json.LastName}}`
    - `loyalty_tier` ← `{{$json.Loyalty_Tier__c}}`
    - `lifetime_value` ← `{{$json.Lifetime_Value__c}}`
    - `signup_date` ← `{{$json.CreatedDate}}`
- **Key expressions or variables used:** Salesforce source fields from trigger output.
- **Input and output connections:**  
  - Input: `Salesforce Trigger`
  - Output: `Enrich by Email`
- **Version-specific requirements:** Uses Set node version 3.4.
- **Edge cases or potential failure types:**  
  - Missing `Email` will likely break enrichment usefulness
  - Missing custom fields like `Loyalty_Tier__c` or `Lifetime_Value__c` produce null/empty values
  - Name concatenation may produce leading/trailing spaces if one part is missing
- **Sub-workflow reference:** None.

#### Sticky Note - Header
- **Type and technical role:** Sticky note; documents workflow purpose and setup prerequisites.
- **Configuration choices:** Highlights required accounts, required Salesforce custom fields, testing advice, and customization areas.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** None.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

#### Sticky Note
- **Type and technical role:** Sticky note; describes Step 1 behavior.
- **Configuration choices:** Documents Salesforce trigger purpose and captured fields.
- **Input and output connections:** None.

#### Sticky Note12
- **Type and technical role:** Sticky note; describes Step 2 behavior.
- **Configuration choices:** Documents field mapping and downstream role.
- **Input and output connections:** None.

---

## Block 2: Creator Profile Enrichment

### Overview
This block sends the contact email to Influencers Club and retrieves creator profile data from social platforms. It transforms a CRM contact into a richer creator candidate profile for AI analysis.

### Nodes Involved
- Enrich by Email
- Step 2 Header1

### Node Details

#### Enrich by Email
- **Type and technical role:** `n8n-nodes-influencersclub.influencersClub`; creator enrichment lookup by email.
- **Configuration choices:**  
  - Email input: `{{$json.email}}`
  - Uses Influencers Club API credentials
- **Key expressions or variables used:** `{{$json.email}}`
- **Input and output connections:**  
  - Input: `Extract Data Fields`
  - Output: `AI Creator Classification Agent4`
- **Version-specific requirements:** Uses node version 1. Requires the Influencers Club community/custom node to be installed in n8n.
- **Edge cases or potential failure types:**  
  - Invalid or missing API credentials
  - No creator match for the email
  - API rate limits or transient HTTP failures
  - Partial platform data causing weaker AI classification
- **Sub-workflow reference:** None.

#### Step 2 Header1
- **Type and technical role:** Sticky note; describes enrichment API behavior.
- **Configuration choices:** Notes expected endpoint behavior and need for API key configuration.
- **Input and output connections:** None.

---

## Block 3: AI Creator Classification

### Overview
This block uses GPT-4o-mini through a LangChain agent to classify the enriched creator profile into structured partnership intelligence. A strict structured output parser ensures the result conforms to a predefined schema.

### Nodes Involved
- AI Creator Classification Agent4
- OpenAI Classifier Model4
- Classification Output Parser4
- Sticky Note14
- Sticky Note1

### Node Details

#### OpenAI Classifier Model4
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`; language model backend for classification.
- **Configuration choices:**  
  - Model: `gpt-4o-mini`
  - Temperature: `0.3` for more deterministic outputs
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  - AI language model input to `AI Creator Classification Agent4`
- **Version-specific requirements:** Node version 1.3. Requires OpenAI credentials.
- **Edge cases or potential failure types:**  
  - OpenAI auth failure
  - Token limits if enrichment payload is unexpectedly large
  - Model availability or quota exhaustion
- **Sub-workflow reference:** None.

#### Classification Output Parser4
- **Type and technical role:** `@n8n/n8n-nodes-langchain.outputParserStructured`; validates and structures model output.
- **Configuration choices:**  
  - Manual JSON schema
  - Requires top-level `classification`
  - Enforces fields like:
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
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  - Connected as output parser to `AI Creator Classification Agent4`
- **Version-specific requirements:** Version 1.2.
- **Edge cases or potential failure types:**  
  - LLM returns invalid JSON
  - Enum mismatch, missing required fields, or type mismatch
  - Numeric values outside schema constraints
- **Sub-workflow reference:** None.

#### AI Creator Classification Agent4
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`; orchestrates classification prompt, model, and parser.
- **Configuration choices:**  
  - Input text: `={{ JSON.stringify($json) }}`
  - Prompt instructs the model to:
    - pick a primary platform
    - classify tier strictly by follower count
    - identify niche and subcategory
    - calculate value score
    - assign brand fit score
    - extract themes
    - infer audience demographics
    - provide reasoning
  - Enforces strict JSON output only
- **Key expressions or variables used:** `{{ JSON.stringify($json) }}`
- **Input and output connections:**  
  - Input: `Enrich by Email`
  - AI model: `OpenAI Classifier Model4`
  - AI output parser: `Classification Output Parser4`
  - Output: `Route Logic`
- **Version-specific requirements:** Version 3.1.
- **Edge cases or potential failure types:**  
  - Schema-valid but semantically poor classifications
  - Empty or sparse enrichment data leading to weak inference
  - Prompt/model drift if fields from enrichment node change shape
- **Sub-workflow reference:** None.

#### Sticky Note14
- **Type and technical role:** Sticky note; documents classification objectives and expected outputs.
- **Configuration choices:** Explains eight classification dimensions and expected processing time.
- **Input and output connections:** None.

#### Sticky Note1
- **Type and technical role:** Sticky note; high-level process note with external explanation link.
- **Configuration choices:** Includes link: [Full explanation](https://influencers.club/creatorbook/find-creators-inside-loyalty-program/)
- **Input and output connections:** None.

---

## Block 4: Outreach Routing Logic

### Overview
This block translates AI classification into an actionable outreach strategy. It determines whether the contact should enter an ambassador program, receive surprise rewards, get niche-specific perks, or remain in standard nurture.

### Nodes Involved
- Route Logic
- Sticky Note15

### Node Details

#### Route Logic
- **Type and technical role:** `n8n-nodes-base.code`; custom JavaScript decision engine.
- **Configuration choices:**  
  - Reads either:
    - `item.json.output.classification`, or
    - `item.json.classification`
  - If classification is missing, emits an error object instead of failing outright
  - Strategy rules:
    - `ambassador_program` for `micro`, `mid`, `macro`
    - `surprise_rewards` for `nano` with engagement `>= 3`
    - `niche_specific_perks` for selected e-commerce niches with engagement `>= 1`
    - otherwise `standard_outreach`
  - Builds:
    - `routing.strategy`
    - `routing.reason`
    - `routing.recommended_perks`
    - `routing.next_action`
    - `routing.requires_personalization`
    - `routing.priority_level`
  - Also creates `profile_data`
- **Key expressions or variables used:** Internal JS variables:
  - `classification`
  - `tier`
  - `engagement`
  - `niche`
  - `followers`
  - niche mapping object
- **Input and output connections:**  
  - Input: `AI Creator Classification Agent4`
  - Outputs to:
    - `AI Creator Classification Agent5`
    - `Merge2`
- **Version-specific requirements:** Code node version 2.
- **Edge cases or potential failure types:**  
  - `classification.username.replace(...)` can fail if `username` is null or undefined
  - `followers.toLocaleString()` can fail if `followers` is missing or non-numeric
  - If parser output shape changes, routing may default to error records
- **Sub-workflow reference:** None.

#### Sticky Note15
- **Type and technical role:** Sticky note; documents the 4 routing strategies.
- **Configuration choices:** Explains criteria, perks, next action, and output enrichment.
- **Input and output connections:** None.

---

## Block 5: AI Personalization Generation

### Overview
This block generates customized creator-facing messaging based on the routing and profile context. It produces email content plus optional in-app and SMS messaging in structured JSON.

### Nodes Involved
- AI Creator Classification Agent5
- OpenAI Classifier Model5
- Classification Output Parser5
- Sticky Note16

### Node Details

#### OpenAI Classifier Model5
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`; language model backend for personalization.
- **Configuration choices:**  
  - Model: `gpt-4o-mini`
  - Temperature: `0.3`
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  - AI language model input to `AI Creator Classification Agent5`
- **Version-specific requirements:** Version 1.3.
- **Edge cases or potential failure types:** Same OpenAI-related issues as the classification model node.

#### Classification Output Parser5
- **Type and technical role:** `@n8n/n8n-nodes-langchain.outputParserStructured`; validates personalization output.
- **Configuration choices:**  
  - Manual schema with top-level `personalization`
  - Required fields:
    - `email_subject`
    - `email_body`
    - `in_app_message`
    - `sms_message`
    - `personalization_notes`
    - `follow_up_timing`
    - `content_hooks`
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  - Connected as output parser to `AI Creator Classification Agent5`
- **Version-specific requirements:** Version 1.2.
- **Edge cases or potential failure types:**  
  - LLM output invalid against schema
  - Length instructions are prompt-only, not schema-enforced
  - `sms_message` can be string or null, which is acceptable
- **Sub-workflow reference:** None.

#### AI Creator Classification Agent5
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`; AI personalization generator.
- **Configuration choices:**  
  - Input text: `={{ JSON.stringify($json) }}`
  - System message defines:
    - recognition-first tone
    - no generic templates
    - no use of “influencer”
    - no posting requests
    - strategy-specific tone
    - valid JSON only
- **Key expressions or variables used:** `{{ JSON.stringify($json) }}`
- **Input and output connections:**  
  - Input: `Route Logic`
  - AI model: `OpenAI Classifier Model5`
  - AI output parser: `Classification Output Parser5`
  - Output: `Merge2`
- **Version-specific requirements:** Version 3.1.
- **Edge cases or potential failure types:**  
  - Content may be valid JSON but not aligned with actual route due to insufficient prompt context
  - If incoming JSON lacks expected routing/profile fields, personalization quality declines
- **Sub-workflow reference:** None.

#### Sticky Note16
- **Type and technical role:** Sticky note; documents the expected personalization outputs and tone rules.
- **Configuration choices:** Covers email, in-app, SMS, hooks, and timing.
- **Input and output connections:** None.

---

## Block 6: Merge and Ambassador Activation

### Overview
This block combines the routing result and the personalization result, then determines a formal ambassador activation tier. It is the decision point that converts marketing intelligence into CRM program status.

### Nodes Involved
- Merge2
- Activation
- Sticky Note18

### Node Details

#### Merge2
- **Type and technical role:** `n8n-nodes-base.merge`; combines two branches.
- **Configuration choices:**  
  - Mode: `combine`
  - Combine By: `combineAll`
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  - Input 0: `Route Logic`
  - Input 1: `AI Creator Classification Agent5`
  - Output: `Activation`
- **Version-specific requirements:** Version 3.2.
- **Edge cases or potential failure types:**  
  - If one branch fails or returns mismatched item counts, merged output may not align as expected
  - Combine behavior must be verified when scaling to multiple simultaneous items
- **Sub-workflow reference:** None.

#### Activation
- **Type and technical role:** `n8n-nodes-base.code`; JavaScript rules engine for ambassador tier assignment.
- **Configuration choices:**  
  - Reads:
    - `classification`
    - `routing`
    - `personalization`
  - Computes:
    - `activation.level`
    - `activation.ambassador_tier`
    - `activation.actions`
    - `activation.routing_output`
  - Decision rules:
    - `elite_ambassador` if macro/mid and high engagement/fit/value
    - `core_ambassador` if micro/mid and good engagement/fit/value
    - `rising_star` if nano high engagement or micro high value, plus fit threshold
    - `standard_ambassador` if `r.strategy_type === 'ambassador'`
    - otherwise `none`
- **Key expressions or variables used:** Internal JS variables:
  - `c`, `r`, `p`
  - `tier`, `eng`, `fit`, `val`
- **Input and output connections:**  
  - Input: `Merge2`
  - Output: `Switch`
- **Version-specific requirements:** Code node version 2.
- **Edge cases or potential failure types:**  
  - There is a logic inconsistency: `Route Logic` sets `routing.strategy`, but this node checks `r.strategy_type === 'ambassador'`
  - Because of that mismatch, the `standard_ambassador` branch will likely never be reached
  - Missing classification values default to zero and may suppress activation unexpectedly
- **Sub-workflow reference:** None.

#### Sticky Note18
- **Type and technical role:** Sticky note; documents activation thresholds and intended program tiers.
- **Configuration choices:** Describes Elite, Core, Rising Star, and Standard logic.
- **Input and output connections:** None.

---

## Block 7: Tier Routing and Salesforce CRM Update

### Overview
This block routes activation outcomes through a Switch node and updates the originating Salesforce contact with creator, activation, and outreach metadata. Although there are multiple switch outputs, all current branches lead to the same Salesforce update node.

### Nodes Involved
- Switch
- Update a contact
- Sticky Note19
- Sticky Note20
- Sticky Note - Setup

### Node Details

#### Switch
- **Type and technical role:** `n8n-nodes-base.switch`; branch selector based on activation routing output.
- **Configuration choices:**  
  - Mode: expression
  - Output expression:
    - `elite -> 0`
    - `core -> 1`
    - `rising -> 2`
    - `standard -> 3`
    - `no_activation -> 3`
    - fallback `3`
- **Key expressions or variables used:**  
  `={{ {elite:0,core:1,rising:2,standard:3,no_activation:3}[$json.activation?.routing_output] ?? 3 }}`
- **Input and output connections:**  
  - Input: `Activation`
  - Outputs 0,1,2,3 all connect to `Update a contact`
- **Version-specific requirements:** Version 3.
- **Edge cases or potential failure types:**  
  - Since all outputs currently converge to the same node, the switch has no operational branching effect yet
  - Future path customization must ensure output indexes remain aligned
- **Sub-workflow reference:** None.

#### Update a contact
- **Type and technical role:** `n8n-nodes-base.salesforce`; updates the Salesforce contact record with custom creator metadata.
- **Configuration choices:**  
  - Resource: `contact`
  - Operation: `update`
  - Contact ID: `={{ $('Salesforce Trigger').first().json.Id }}`
  - Updates custom fields:
    - `Is_Creator__c`
    - `Creator_Tier__c`
    - `Creator_Niche__c`
    - `Creator_Subcategory__c`
    - `Primary_Platform__c`
    - `Follower_Count__c`
    - `Engagement_Rate__c`
    - `Social_Username__c`
    - `Profile_URL__c`
    - `Value_Score__c`
    - `Brand_Fit_Score__c`
    - `Content_Themes__c`
    - `Audience_Demographics__c`
    - `Ambassador_Tier__c`
    - `Activation_Level__c`
    - `Activation_Date__c`
    - `Next_Activation_Action__c`
    - `Outreach_Strategy__c`
    - `Outreach_Sent_Date__c`
- **Key expressions or variables used:**  
  Examples:
  - `{{$json.activation?.level !== 'none'}}`
  - `{{$json.classification?.tier_name || ''}}`
  - `{{($json.classification?.content_themes || []).join(', ')}}`
  - `{{$now.toISO()}}`
- **Input and output connections:**  
  - Input: all four outputs from `Switch`
  - Output: `Send Email`
- **Version-specific requirements:** Version 1.
- **Edge cases or potential failure types:**  
  - Custom fields must exist exactly as referenced
  - Salesforce data type mismatches may fail updates
  - `Outreach_Strategy__c` is mapped from `classification?.routing?.strategy`, but routing actually exists at top-level `routing`, not under `classification`
  - Therefore `Outreach_Strategy__c` will likely remain blank unless corrected to `$json.routing?.strategy`
  - `contactId` depends on original trigger item; if item ordering breaks in complex runs, wrong contact updates are possible
- **Sub-workflow reference:** None.

#### Sticky Note19
- **Type and technical role:** Sticky note; explains the intended tier-based routing behavior.
- **Configuration choices:** Documents intended different handling workflows for switch outputs.
- **Input and output connections:** None.

#### Sticky Note20
- **Type and technical role:** Sticky note; explains Salesforce update purpose and field categories.
- **Configuration choices:** Documents creator classification, social metrics, partnership value, ambassador status, and tracking fields.
- **Input and output connections:** None.

#### Sticky Note - Setup
- **Type and technical role:** Sticky note; Salesforce custom field checklist.
- **Configuration choices:** Lists required custom fields on Contact.
- **Input and output connections:** None.

---

## Block 8: Optional Personalized Email Sending

### Overview
This final block is intended to send a personalized email through SendGrid after the Salesforce update. It is currently disabled, so the workflow will stop after the CRM update unless the node is manually enabled.

### Nodes Involved
- Send Email
- Sticky Note21

### Node Details

#### Send Email
- **Type and technical role:** `n8n-nodes-base.sendGrid`; transactional email sender.
- **Configuration choices:**  
  - Node is **disabled**
  - Subject:
    `={{ $('Switch').item.json.output?.personalization?.email_subject || 'Partnership Opportunity' }}`
  - To:
    `={{ $('Extract Data Fields').first().json.email }}`
  - From name: `Partnership Team`
  - From email: `user@example.com`
  - Content:
    `={{ $('Switch').first().json.output?.personalization?.email_body || 'Thank you for being a valued customer!' }}`
- **Key expressions or variables used:** References data via `Switch` and `Extract Data Fields`.
- **Input and output connections:**  
  - Input: `Update a contact`
  - No downstream nodes
- **Version-specific requirements:** Version 1. Requires SendGrid API credentials and a verified sender identity.
- **Edge cases or potential failure types:**  
  - Disabled node means no email is sent
  - Expressions likely reference the wrong data shape:
    - personalization is not under `output` after `Activation`
    - more reliable reference would likely be current item JSON or `Activation` output
  - `fromEmail` must be a verified SendGrid sender
  - Bounce or suppression list issues may prevent delivery
- **Sub-workflow reference:** None.

#### Sticky Note21
- **Type and technical role:** Sticky note; describes intended personalized email behavior.
- **Configuration choices:** Notes use of a human sender identity and tracking.
- **Input and output connections:** None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note - Header | stickyNote | Workflow setup and prerequisites documentation |  |  | ## 🎯 Discover Creators Among Loyalty Program Customers and Activate Tiered Ambassador Programs<br>### Setup steps<br>**Required accounts:**<br>- Salesforce (with loyalty_tier custom field)<br>- influencers.club API key<br>- OpenAI API key (GPT-4o-mini)<br>- SendGrid account<br>**Before running:**<br>1. Create loyalty_tier custom field in Salesforce<br>2. Add 20+ custom creator fields to Salesforce Contact object<br>3. Configure Salesforce trigger<br>4. Add your API keys to credentials<br>5. Test with 1-2 contacts before scaling<br>**Customize:**<br>- Adjust tier thresholds in ambassador activation logic<br>- Modify routing strategies for your programs<br>- Update AI prompts for your brand voice |
| Sticky Note - Setup | stickyNote | Salesforce custom field checklist |  |  | **📋 SALESFORCE SETUP CHECKLIST**<br>Create these custom fields on the Contact object before running:<br>✅ **Creator Info:** Is_Creator__c, Creator_Tier__c, Creator_Niche__c, Creator_Subcategory__c, Primary_Platform__c<br>✅ **Metrics:** Follower_Count__c, Engagement_Rate__c, Social_Username__c, Profile_URL__c, Creator_Bio__c<br>✅ **Value:** Value_Score__c, Brand_Fit_Score__c, Content_Themes__c, Audience_Demographics__c<br>✅ **Ambassador:** Ambassador_Tier__c, Activation_Level__c, Activation_Date__c, Next_Activation_Action__c<br>✅ **Tracking:** Outreach_Strategy__c, Outreach_Priority__c, Outreach_Sent_Date__c, Personalization_Used__c |
| Salesforce Trigger | salesforceTrigger | Entry point on new Salesforce contact creation |  | Extract Data Fields | ## STEP 1: Trigger on Contact Created<br>Monitors Salesforce for newly created contacts and fires immediately. Captures contact ID, email, name, loyalty tier, and other relevant fields for downstream processing. |
| Extract Data Fields | set | Normalize Salesforce contact data | Salesforce Trigger | Enrich by Email | ## STEP 2: Extract Contact Details<br>Maps Salesforce contact fields to a clean data object including email address (required for enrichment), name, loyalty tier, lifetime value, and signup date. Provides full context for downstream processing. |
| Enrich by Email | influencersClub | Enrich contact into creator/social profile by email | Extract Data Fields | AI Creator Classification Agent4 | ##  STEP 3: Enrich Creator Profile<br>**API Call to influencers.club**<br>- Uses `/public/v1/creators/enrich/email/`<br>- Input: Creator email from Salesforce contact<br>- Output: Complete main-platform profile<br>**Configure:**<br>1. Add your API key to HTTP Header Auth<br>2. Error handling sends to fallback flow |
| OpenAI Classifier Model4 | lmChatOpenAi | LLM backend for creator classification |  | AI Creator Classification Agent4 | ##  STEP 5: AI Creator Classification Agent<br>**What happens here:**<br>AI agent analyzes enriched social data and classifies the creator across 8 dimensions for partnership intelligence.<br>**Classification outputs:**<br>1. **Primary Platform:** Instagram/TikTok/YouTube/Twitter (highest engagement)<br>2. **Influencer Tier:** Nano (<10K) \| Micro (10K-100K) \| Mid (100K-500K) \| Macro (500K+)<br>3. **Niche Detection:** Fashion, Beauty, Fitness, Food, Pets, Parenting, Tech, etc. with specific subcategories (e.g., Muay Thai training, sustainable fashion)<br>4. **Value Score (0-100):** Calculated from tier + engagement + niche relevance + multi-platform + professional signals<br>5. **Brand Fit Score (1-10):** Content quality, audience alignment, collaboration signals<br>6. **Content Themes:** 3-5 specific recurring topics from bio (not just niche)<br>7. **Audience Demographics:** Inferred gender, age range, interests<br>8. **Reasoning:** 2-3 sentence explanation of classification decisions<br>**Example output:**<br>Tier: Micro \| Niche: Fitness > Muay Thai training \| Value: 65 \| Fit: 8<br>**Processing time:** 2-4 seconds |
| Classification Output Parser4 | outputParserStructured | Structured schema enforcement for classification output |  | AI Creator Classification Agent4 | ##  STEP 5: AI Creator Classification Agent<br>**What happens here:**<br>AI agent analyzes enriched social data and classifies the creator across 8 dimensions for partnership intelligence.<br>**Classification outputs:**<br>1. **Primary Platform:** Instagram/TikTok/YouTube/Twitter (highest engagement)<br>2. **Influencer Tier:** Nano (<10K) \| Micro (10K-100K) \| Mid (100K-500K) \| Macro (500K+)<br>3. **Niche Detection:** Fashion, Beauty, Fitness, Food, Pets, Parenting, Tech, etc. with specific subcategories (e.g., Muay Thai training, sustainable fashion)<br>4. **Value Score (0-100):** Calculated from tier + engagement + niche relevance + multi-platform + professional signals<br>5. **Brand Fit Score (1-10):** Content quality, audience alignment, collaboration signals<br>6. **Content Themes:** 3-5 specific recurring topics from bio (not just niche)<br>7. **Audience Demographics:** Inferred gender, age range, interests<br>8. **Reasoning:** 2-3 sentence explanation of classification decisions<br>**Example output:**<br>Tier: Micro \| Niche: Fitness > Muay Thai training \| Value: 65 \| Fit: 8<br>**Processing time:** 2-4 seconds |
| AI Creator Classification Agent4 | agent | Generate structured creator classification | Enrich by Email; OpenAI Classifier Model4; Classification Output Parser4 | Route Logic | ##  STEP 5: AI Creator Classification Agent<br>**What happens here:**<br>AI agent analyzes enriched social data and classifies the creator across 8 dimensions for partnership intelligence.<br>**Classification outputs:**<br>1. **Primary Platform:** Instagram/TikTok/YouTube/Twitter (highest engagement)<br>2. **Influencer Tier:** Nano (<10K) \| Micro (10K-100K) \| Mid (100K-500K) \| Macro (500K+)<br>3. **Niche Detection:** Fashion, Beauty, Fitness, Food, Pets, Parenting, Tech, etc. with specific subcategories (e.g., Muay Thai training, sustainable fashion)<br>4. **Value Score (0-100):** Calculated from tier + engagement + niche relevance + multi-platform + professional signals<br>5. **Brand Fit Score (1-10):** Content quality, audience alignment, collaboration signals<br>6. **Content Themes:** 3-5 specific recurring topics from bio (not just niche)<br>7. **Audience Demographics:** Inferred gender, age range, interests<br>8. **Reasoning:** 2-3 sentence explanation of classification decisions<br>**Example output:**<br>Tier: Micro \| Niche: Fitness > Muay Thai training \| Value: 65 \| Fit: 8<br>**Processing time:** 2-4 seconds |
| Route Logic | code | Translate classification into outreach strategy | AI Creator Classification Agent4 | AI Creator Classification Agent5; Merge2 | ##  STEP 6: Intelligent Routing Logic<br>**4 Routing Strategies:**<br>**Route 1: Ambassador Program**<br>- Criteria: Micro/Mid/Macro tier<br>- Perks: Discount codes, monthly products, commissions, featured placement, early access<br>- Next: Send formal ambassador invitation<br>**Route 2: Surprise Rewards**<br>- Criteria: Nano tier + ≥3% engagement<br>- Perks: Gift package, thank-you note, early access, personalized selection<br>- Next: Send surprise (no ask required)<br>**Route 3: Niche-Specific Perks**<br>- Criteria: E-commerce niche + ≥1% engagement<br>- Perks: Customized by niche (fitness: gear packages, beauty: bundles, fashion: styling sessions, etc.)<br>- Next: Send niche perk offer<br>**Route 4: Standard Outreach**<br>- Criteria: All others<br>- Perks: Newsletter, promotions, community, loyalty points<br>- Next: General nurture sequence<br>**Output enrichment:**<br>Adds routing strategy, reason, perks, next action, priority level to data object |
| OpenAI Classifier Model5 | lmChatOpenAi | LLM backend for personalization generation |  | AI Creator Classification Agent5 | ##  STEP 7: AI-Powered Message Personalization<br>**Generated content:**<br>1. **Email Subject:** Contextual, references achievements, avoids marketing language (40-60 chars)<br>2. **Email Body:** Recognition of content themes, why they align with brand, tier-appropriate perks, soft next step<br>3. **In-App Message:** Teaser that drives email open<br>4. **SMS (optional):**  High-priority only<br>5. **Content Hooks:** 3-5 themes for follow-ups<br>6. **Follow-up Timing:** Recommended wait (7/14/30 days)<br>**Tone by strategy:**<br>- Ambassador: Warm, earned opportunity<br>- Surprise: Delight + appreciation<br>- Niche Perk: Collaborative, content-first<br>**Never mentions:**<br>- Follower count mechanically<br>- Request to post<br>- Generic influencer language<br>- Collaboration/partnership (too transactional) |
| Classification Output Parser5 | outputParserStructured | Structured schema enforcement for personalization output |  | AI Creator Classification Agent5 | ##  STEP 7: AI-Powered Message Personalization<br>**Generated content:**<br>1. **Email Subject:** Contextual, references achievements, avoids marketing language (40-60 chars)<br>2. **Email Body:** Recognition of content themes, why they align with brand, tier-appropriate perks, soft next step<br>3. **In-App Message:** Teaser that drives email open<br>4. **SMS (optional):**  High-priority only<br>5. **Content Hooks:** 3-5 themes for follow-ups<br>6. **Follow-up Timing:** Recommended wait (7/14/30 days)<br>**Tone by strategy:**<br>- Ambassador: Warm, earned opportunity<br>- Surprise: Delight + appreciation<br>- Niche Perk: Collaborative, content-first<br>**Never mentions:**<br>- Follower count mechanically<br>- Request to post<br>- Generic influencer language<br>- Collaboration/partnership (too transactional) |
| AI Creator Classification Agent5 | agent | Generate personalized outreach messages | Route Logic; OpenAI Classifier Model5; Classification Output Parser5 | Merge2 | ##  STEP 7: AI-Powered Message Personalization<br>**Generated content:**<br>1. **Email Subject:** Contextual, references achievements, avoids marketing language (40-60 chars)<br>2. **Email Body:** Recognition of content themes, why they align with brand, tier-appropriate perks, soft next step<br>3. **In-App Message:** Teaser that drives email open<br>4. **SMS (optional):**  High-priority only<br>5. **Content Hooks:** 3-5 themes for follow-ups<br>6. **Follow-up Timing:** Recommended wait (7/14/30 days)<br>**Tone by strategy:**<br>- Ambassador: Warm, earned opportunity<br>- Surprise: Delight + appreciation<br>- Niche Perk: Collaborative, content-first<br>**Never mentions:**<br>- Follower count mechanically<br>- Request to post<br>- Generic influencer language<br>- Collaboration/partnership (too transactional) |
| Merge2 | merge | Combine routing data and personalization data | Route Logic; AI Creator Classification Agent5 | Activation |  |
| Activation | code | Assign ambassador activation level and actions | Merge2 | Switch | ## STEP 8: Ambassador Activation<br>Assigns creators to tiered programs based on engagement, brand fit, and value: Elite (macro/mid, 5%+ engagement, 15% commission), Core (micro/mid, 4%+ engagement, 12% commission), Rising Star (nano 5%+ or micro 75+ value), Standard (basic program). |
| Switch | switch | Map activation outcome to branch index | Activation | Update a contact | ## STEP 9: Route by Tier<br>Switch node routes by ambassador tier: Elite (0) → Manual review. Core (1) → Automated monitoring. Rising (2) → Full automation. Standard/None (3) → Pure automation. Each path should get appropriate handling workflow. |
| Update a contact | salesforce | Write creator and activation fields back to Salesforce | Switch | Send Email | ## STEP 10: Update Salesforce CRM<br>Writes 20+ custom fields to the Salesforce Contact record: creator classification (tier, niche, platform), social metrics (followers, engagement, username), partnership value (scores, themes), ambassador status (tier, activation date), and outreach tracking. Enables segmentation and analytics.<br>**📋 SALESFORCE SETUP CHECKLIST**<br>Create these custom fields on the Contact object before running:<br>✅ **Creator Info:** Is_Creator__c, Creator_Tier__c, Creator_Niche__c, Creator_Subcategory__c, Primary_Platform__c<br>✅ **Metrics:** Follower_Count__c, Engagement_Rate__c, Social_Username__c, Profile_URL__c, Creator_Bio__c<br>✅ **Value:** Value_Score__c, Brand_Fit_Score__c, Content_Themes__c, Audience_Demographics__c<br>✅ **Ambassador:** Ambassador_Tier__c, Activation_Level__c, Activation_Date__c, Next_Activation_Action__c<br>✅ **Tracking:** Outreach_Strategy__c, Outreach_Priority__c, Outreach_Sent_Date__c, Personalization_Used__c |
| Send Email | sendGrid | Send personalized outreach email | Update a contact |  | ## STEP 11: Send Personalized Email<br>Delivers AI-generated, tier-appropriate email via SendGrid. Uses real person as sender (not marketing@) for higher open rates. Includes personalized subject, recognition-first body copy, and soft CTA. Tracks opens, clicks, bounces automatically. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create the Salesforce trigger node**
   - Add a `Salesforce Trigger` node.
   - Configure Salesforce credentials.
   - Set trigger event to `Contact Created`.
   - Set polling to every minute.
   - Confirm the connected Salesforce user can read Contact records.

2. **Create the field extraction node**
   - Add a `Set` node named `Extract Data Fields`.
   - Connect `Salesforce Trigger -> Extract Data Fields`.
   - Add these fields:
     - `email` as string: `{{$json.Email}}`
     - `member_name` as string: `{{$json.FirstName}} {{$json.LastName}}`
     - `loyalty_tier` as string: `{{$json.Loyalty_Tier__c}}`
     - `lifetime_value` as number: `{{$json.Lifetime_Value__c}}`
     - `signup_date` as string: `{{$json.CreatedDate}}`

3. **Install or enable the Influencers Club node**
   - Ensure the `n8n-nodes-influencersclub.influencersClub` package is available in your n8n instance.
   - Add the node and name it `Enrich by Email`.
   - Connect `Extract Data Fields -> Enrich by Email`.
   - Set the email parameter to `{{$json.email}}`.
   - Add Influencers Club API credentials.

4. **Create the OpenAI model node for classification**
   - Add an `OpenAI Chat Model` LangChain node.
   - Name it `OpenAI Classifier Model4`.
   - Choose model `gpt-4o-mini`.
   - Set temperature to `0.3`.
   - Add OpenAI credentials.

5. **Create the structured output parser for classification**
   - Add a `Structured Output Parser` LangChain node.
   - Name it `Classification Output Parser4`.
   - Set schema mode to manual.
   - Paste a schema that requires a top-level object with `classification`.
   - Include fields for:
     - primary platform
     - tier and tier name
     - niche and niche name
     - niche subcategory
     - followers
     - engagement rate
     - username
     - profile URL
     - value score
     - content themes
     - audience demographics
     - brand fit score
     - reasoning

6. **Create the AI agent for classification**
   - Add a LangChain `Agent` node named `AI Creator Classification Agent4`.
   - Connect:
     - `Enrich by Email -> AI Creator Classification Agent4`
     - `OpenAI Classifier Model4 -> AI Creator Classification Agent4` through the AI language model port
     - `Classification Output Parser4 -> AI Creator Classification Agent4` through the AI output parser port
   - Set input text to `{{ JSON.stringify($json) }}`.
   - Use a system prompt that:
     - analyzes creator platforms
     - chooses the primary platform
     - classifies tier strictly from follower count
     - assigns niche/subcategory
     - calculates value score
     - assigns brand fit score
     - extracts 3–5 themes
     - infers audience demographics
     - returns valid JSON only

7. **Create the routing code node**
   - Add a `Code` node named `Route Logic`.
   - Connect `AI Creator Classification Agent4 -> Route Logic`.
   - Paste logic that:
     - reads the `classification` object
     - assigns one of these strategies:
       - `ambassador_program`
       - `surprise_rewards`
       - `niche_specific_perks`
       - `standard_outreach`
     - adds perks, reason, next action, personalization flag, and priority level
     - builds a simplified `profile_data` object
   - Recommended fix while rebuilding:
     - protect against missing username with something like:
       `const safeUsername = classification.username || '';`
     - then derive the name safely

8. **Create the OpenAI model node for personalization**
   - Add another `OpenAI Chat Model` LangChain node.
   - Name it `OpenAI Classifier Model5`.
   - Choose `gpt-4o-mini`.
   - Set temperature to `0.3`.
   - Reuse the same OpenAI credentials if desired.

9. **Create the structured output parser for personalization**
   - Add another `Structured Output Parser` node.
   - Name it `Classification Output Parser5`.
   - Define a schema with a top-level `personalization` object.
   - Required fields:
     - `email_subject`
     - `email_body`
     - `in_app_message`
     - `sms_message`
     - `personalization_notes`
     - `follow_up_timing`
     - `content_hooks`

10. **Create the AI personalization agent**
    - Add a LangChain `Agent` node named `AI Creator Classification Agent5`.
    - Connect:
      - `Route Logic -> AI Creator Classification Agent5`
      - `OpenAI Classifier Model5 -> AI Creator Classification Agent5`
      - `Classification Output Parser5 -> AI Creator Classification Agent5`
    - Set input text to `{{ JSON.stringify($json) }}`.
    - Use a prompt that:
      - prioritizes recognition over selling
      - avoids the word “influencer”
      - avoids posting requests
      - outputs email, in-app, optional SMS, hooks, and timing
      - returns JSON only

11. **Create the merge node**
    - Add a `Merge` node named `Merge2`.
    - Set mode to `Combine`.
    - Set combine behavior to `Combine All`.
    - Connect:
      - `Route Logic -> Merge2` as input 1
      - `AI Creator Classification Agent5 -> Merge2` as input 2

12. **Create the activation code node**
    - Add a `Code` node named `Activation`.
    - Connect `Merge2 -> Activation`.
    - Implement logic that:
      - reads classification, routing, and personalization
      - sets activation to:
        - `elite_ambassador`
        - `core_ambassador`
        - `rising_star`
        - `standard_ambassador`
        - or none
      - writes `activation.level`, `activation.ambassador_tier`, `activation.actions`, and `activation.routing_output`
    - Recommended fix while rebuilding:
      - use `r.strategy === 'ambassador_program'` instead of `r.strategy_type === 'ambassador'`

13. **Create the switch node**
    - Add a `Switch` node named `Switch`.
    - Connect `Activation -> Switch`.
    - Use expression mode.
    - Set output expression to:
      `={{ {elite:0,core:1,rising:2,standard:3,no_activation:3}[$json.activation?.routing_output] ?? 3 }}`
    - Create four outputs.

14. **Create the Salesforce update node**
    - Add a `Salesforce` node named `Update a contact`.
    - Set:
      - Resource: `Contact`
      - Operation: `Update`
      - Contact ID: `{{$('Salesforce Trigger').first().json.Id}}`
    - Connect all four outputs of `Switch` to `Update a contact`.
    - Add custom field mappings:
      - `Is_Creator__c` ← `{{$json.activation?.level !== 'none'}}`
      - `Creator_Tier__c` ← `{{$json.classification?.tier_name || ''}}`
      - `Creator_Niche__c` ← `{{$json.classification?.niche_name || ''}}`
      - `Creator_Subcategory__c` ← `{{$json.classification?.niche_subcategory || ''}}`
      - `Primary_Platform__c` ← `{{$json.classification?.primary_platform || ''}}`
      - `Follower_Count__c` ← `{{$json.classification?.followers || 0}}`
      - `Engagement_Rate__c` ← `{{$json.classification?.engagement_rate || 0}}`
      - `Social_Username__c` ← `{{$json.classification?.username || ''}}`
      - `Profile_URL__c` ← `{{$json.classification?.profile_url || ''}}`
      - `Value_Score__c` ← `{{$json.classification?.value_score || 0}}`
      - `Brand_Fit_Score__c` ← `{{$json.classification?.brand_fit_score || 0}}`
      - `Content_Themes__c` ← `{{($json.classification?.content_themes || []).join(', ')}}`
      - `Audience_Demographics__c` ← `{{$json.classification?.audience_demographics || ''}}`
      - `Ambassador_Tier__c` ← `{{$json.activation?.ambassador_tier || ''}}`
      - `Activation_Level__c` ← `{{$json.activation?.level || ''}}`
      - `Activation_Date__c` ← `{{$now.toISO()}}`
      - `Next_Activation_Action__c` ← `{{($json.activation?.actions || []).join(', ')}}`
      - `Outreach_Strategy__c` ← `{{$json.routing?.strategy || ''}}`
      - `Outreach_Sent_Date__c` ← `{{$now.toISO()}}`
    - Recommended fix:
      - use `$json.routing?.strategy`, not `$json.classification?.routing?.strategy`

15. **Create the SendGrid email node**
    - Add a `SendGrid` node named `Send Email`.
    - Connect `Update a contact -> Send Email`.
    - Configure SendGrid credentials.
    - Set:
      - To email: `{{$('Extract Data Fields').first().json.email}}`
      - From name: a human sender such as `Partnership Team`
      - From email: a verified sender address in SendGrid
      - Subject: use a current-item reference to personalization, for example `{{$json.personalization?.email_subject || 'Partnership Opportunity'}}`
      - Body: `{{$json.personalization?.email_body || 'Thank you for being a valued customer!'}}`
    - If you want to match the provided workflow exactly, disable the node after configuration.
    - Recommended fix:
      - do not reference `output.personalization` unless your upstream node actually nests data that way

16. **Create Salesforce custom fields before testing**
    - On the Salesforce Contact object, create at minimum:
      - `Is_Creator__c`
      - `Creator_Tier__c`
      - `Creator_Niche__c`
      - `Creator_Subcategory__c`
      - `Primary_Platform__c`
      - `Follower_Count__c`
      - `Engagement_Rate__c`
      - `Social_Username__c`
      - `Profile_URL__c`
      - `Creator_Bio__c` if you intend to store bio later
      - `Value_Score__c`
      - `Brand_Fit_Score__c`
      - `Content_Themes__c`
      - `Audience_Demographics__c`
      - `Ambassador_Tier__c`
      - `Activation_Level__c`
      - `Activation_Date__c`
      - `Next_Activation_Action__c`
      - `Outreach_Strategy__c`
      - `Outreach_Priority__c` if you extend the update logic
      - `Outreach_Sent_Date__c`
      - `Personalization_Used__c` if you extend tracking

17. **Configure credentials**
    - **Salesforce:** OAuth2 or supported Salesforce credential with read/update access on Contacts.
    - **OpenAI:** API key authorized for `gpt-4o-mini`.
    - **Influencers Club:** API credential for enrichment.
    - **SendGrid:** API key with mail send permission and verified sender identity.

18. **Test with a small dataset**
    - Create 1–2 Salesforce contacts manually.
    - Ensure they contain valid email addresses likely to resolve in Influencers Club.
    - Verify:
      - enrichment returns profile data
      - AI parser outputs valid structured JSON
      - Salesforce fields update correctly
      - SendGrid node remains disabled until content references are corrected and validated

19. **Optional improvements before production**
    - Add explicit error handling branches after enrichment and AI nodes.
    - Store original personalization content in Salesforce.
    - Separate elite/core/rising paths into different downstream actions.
    - Add rate limiting if many contacts are created at once.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Find and activate creators inside your loyalty program. Step by step workflow to enrich loyalty programme customers emails on Salesforce with multi social (Instagram, Tiktok, Youtube, Twitter, Onlyfans, Twitch and more) and launch personalized emails using the influencer.club API and SendGrid. | [Full explanation](https://influencers.club/creatorbook/find-creators-inside-loyalty-program/) |
| The workflow is currently inactive. | n8n workflow setting: `active: false` |
| The SendGrid node is present but disabled. | Email delivery will not occur unless manually enabled |
| The workflow has a single real entry point. | `Salesforce Trigger` |
| No sub-workflows are invoked. | No Execute Workflow node or workflow-call pattern present |

## Important implementation notes
- There are **two logic mismatches** in the current design:
  1. `Activation` checks `r.strategy_type === 'ambassador'`, but `Route Logic` creates `routing.strategy`, not `strategy_type`.
  2. `Update a contact` writes `Outreach_Strategy__c` from `classification?.routing?.strategy`, but `routing` is top-level.
- The `Send Email` expressions also appear inconsistent with the actual item structure produced upstream.
- For production use, these should be corrected before enabling automated outreach.