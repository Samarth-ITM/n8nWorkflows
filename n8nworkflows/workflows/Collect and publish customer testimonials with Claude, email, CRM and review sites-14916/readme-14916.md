Collect and publish customer testimonials with Claude, email, CRM and review sites

https://n8nworkflows.xyz/workflows/collect-and-publish-customer-testimonials-with-claude--email--crm-and-review-sites-14916


# Collect and publish customer testimonials with Claude, email, CRM and review sites

Good, my analysis is confirmed. Let me now write the comprehensive document. 1. Workflow Overview

**Automated Testimonial & Review Collector** is a multi-stage workflow that automatically collects, AI-processes, and publishes customer testimonials across marketing channels. It operates in two distinct phases:

- **Phase A – Outreach:** When a project is completed (via webhook or daily scheduled batch), the workflow looks up the customer in a CRM, determines whether the optimal follow-up window has been reached, and—if so—generates a personalized Typeform feedback form and sends a review-request email via SendGrid.

- **Phase B – Processing & Publishing:** When a customer submits feedback (via a second webhook), the workflow uses Claude AI (Anthropic) to perform sentiment/quality analysis and generate polished, multi-format testimonials. High-quality testimonials are then formatted for and distributed to WordPress, Trustpilot, and a HubSpot CRM, while all submissions are logged to an Airtable analytics base. A thank-you email closes the loop, and the webhook caller receives a success response.

The workflow contains two independent entry points (a project-completion webhook and a feedback-submission webhook) plus a daily scheduled trigger, making it event-driven and time-aware.

---

### 2. Block-by-Block Analysis

#### Block 1 — Trigger & Customer Data

**Overview:** This block receives the initial signal that a project or purchase has been completed, normalizes the incoming payload, and fetches the full customer profile from HubSpot CRM.

**Nodes Involved:**
- Webhook Trigger - Project Completion
- Daily Batch Check
- Prepare Customer Context
- Fetch Full Customer Profile from CRM

---

**Webhook Trigger - Project Completion**
- **Type:** `n8n-nodes-base.webhook` (v1.1)
- **Technical Role:** HTTP POST entry point. Expects a JSON payload containing `customerId`, `customerEmail`, `customerName`, `projectType`, `completionDate`, `followUpDelay`, and `preferredChannel`.
- **Configuration:** Path `testimonial-trigger`, method `POST`, response mode set to `responseNode` (deferred response until after full processing).
- **Key Expressions:** None in the node itself; downstream nodes reference `$json.body.*`.
- **Input:** External HTTP POST call.
- **Output:** Connects to **Prepare Customer Context**.
- **Edge Cases:** If the external caller does not send a body, downstream expressions fall back gracefully. A missing `customerId` results in a generated `MANUAL-` prefixed ID. Webhook path must be unique on the n8n instance.

---

**Daily Batch Check**
- **Type:** `n8n-nodes-base.scheduleTrigger` (v1.2)
- **Technical Role:** Cron-based scheduled trigger that fires every day at 09:00 UTC.
- **Configuration:** Cron expression `0 9 * * *`.
- **Key Expressions:** None.
- **Input:** None (time-based).
- **Output:** Connects to **Prepare Customer Context**.
- **Edge Cases:** The node emits an empty or minimal item; the downstream Set node must handle missing fields. In production, this trigger should be paired with a CRM query node to fetch a batch of eligible customers (currently not present — the batch mode relies on manual or external data injection).

---

**Prepare Customer Context**
- **Type:** `n8n-nodes-base.set` (v3.4)
- **Technical Role:** Normalizes and merges data from either the webhook body or the schedule trigger into a unified item schema.
- **Configuration — Assignments:**

| Name | Type | Expression |
|---|---|---|
| `customerId` | string | `$json.body?.customerId \|\| $json.customerId \|\| 'MANUAL-' + Date.now()` |
| `customerEmail` | string | `$json.body?.customerEmail \|\| $json.customerEmail` |
| `customerName` | string | `$json.body?.customerName \|\| $json.customerName` |
| `projectType` | string | `$json.body?.projectType \|\| $json.projectType \|\| 'General Service'` |
| `completionDate` | string | `$json.body?.completionDate \|\| $json.completionDate \|\| new Date().toISOString()` |
| `followUpDelay` | number | `$json.body?.followUpDelay \|\| $json.followUpDelay \|\| 7` |
| `preferredChannel` | string | `$json.body?.preferredChannel \|\| $json.preferredChannel \|\| 'email'` |
| `jobId` | string | `'REVIEW-' + Date.now() + '-' + Math.random().toString(36).substr(2, 6).toUpperCase()` |

- **Input:** Webhook Trigger - Project Completion **or** Daily Batch Check.
- **Output:** Connects to **Fetch Full Customer Profile from CRM**.
- **Edge Cases:** If both `body` and root-level fields are missing, `customerId` and `jobId` are auto-generated, `projectType` defaults to "General Service", `followUpDelay` defaults to 7 days. `customerEmail` and `customerName` could be `undefined` — the downstream filter catches missing email.

---

**Fetch Full Customer Profile from CRM**
- **Type:** `n8n-nodes-base.httpRequest` (v4.2)
- **Technical Role:** HTTP GET call to HubSpot CRM to retrieve the contact record for the given `customerId`.
- **Configuration:** URL `https://api.hubspot.com/crm/v3/objects/contacts/{{ $json.customerId }}`, method GET (default), headers `Authorization: Bearer YOUR_TOKEN_HERE` and `Content-Type: application/json`, timeout 10 000 ms.
- **Continue on Fail:** Enabled — if HubSpot returns an error (404, 401, timeout), the workflow continues with empty CRM data.
- **Input:** Prepare Customer Context.
- **Output:** Connects to **Check Optimal Follow-up Timing**.
- **Edge Cases:** 404 if contact does not exist in HubSpot; 401 if token is invalid/expired; the placeholder `YOUR_TOKEN_HERE` must be replaced with a real credential. Because `continueOnFail` is true, the Code node downstream handles missing `properties` gracefully.

---

#### Block 2 — Smart Follow-up & Feedback Collection

**Overview:** This block determines whether enough time has passed since project completion to send a follow-up, filters out customers who have already reviewed or are outside the window, and then creates a personalized Typeform feedback form and dispatches a review-request email.

**Nodes Involved:**
- Check Optimal Follow-up Timing
- Filter Ready for Follow-up
- Generate Personalized Feedback Form
- Send Feedback Request Email

---

**Check Optimal Follow-up Timing**
- **Type:** `n8n-nodes-base.code` (v2)
- **Technical Role:** Custom JavaScript that calculates days since project completion, checks if the customer is within the follow-up window, and enriches the item with structured `customerData`, `timing`, and `projectContext` objects.
- **Configuration:** Mode `runOnceForEachItem`.
- **Key Logic:**
  - `daysSinceCompletion = floor((now - completionDate) / day)`
  - `isReadyForFollowup = daysSinceCompletion >= targetDelay && daysSinceCompletion <= targetDelay + 3`
  - `hasGivenReview` read from HubSpot `testimonial_submitted` property
  - `shouldSendFollowup = isReadyForFollowup && !hasGivenReview`
  - Reads CRM properties: `phone`, `company`, `industry`, `total_purchases`, `lifetime_value`, `last_project_description`, `project_outcome`, `project_deliverables`
- **Input:** Fetch Full Customer Profile from CRM.
- **Output:** Connects to **Filter Ready for Follow-up**.
- **Edge Cases:** If HubSpot data is missing (because the HTTP request failed), `crmData` defaults to `{}`, so phone/company/industry will be `null`, `totalPurchases` defaults to 1, `lifetimeValue` to 0, `hasGivenReview` to `false`. Invalid or unparsable `completionDate` will produce `NaN` for `daysSinceCompletion`, which means `isReadyForFollowup` will be `false`.

---

**Filter Ready for Follow-up**
- **Type:** `n8n-nodes-base.filter` (v2.2)
- **Technical Role:** Passes only items where `timing.shouldSendFollowup` is `true` **and** `customerData.email` exists.
- **Configuration:** Combinator `and`, two conditions:
  1. `$json.timing.shouldSendFollowup` is `true` (boolean)
  2. `$json.customerData.email` exists (string)
- **Input:** Check Optimal Follow-up Timing.
- **Output:** Connects to **Generate Personalized Feedback Form** and **Send Feedback Request Email** (both in parallel).
- **Edge Cases:** Items that fail the filter are silently dropped (no error branch). If the email field is an empty string rather than null/undefined, the "exists" operator may still pass it — consider adding an additional "notEmpty" condition.

---

**Generate Personalized Feedback Form**
- **Type:** `n8n-nodes-base.httpRequest` (v4.2)
- **Technical Role:** HTTP POST to Typeform API to create a feedback form on the fly, personalized with the project type.
- **Configuration:**
  - URL: `https://api.typeform.com/forms`
  - Method: POST
  - Headers: `Authorization: Bearer YOUR_TOKEN_HERE`, `Content-Type: application/json`
  - JSON body (dynamic): Contains a title interpolated with `$json.projectContext.type`, five fields (opinion_scale, two long_text, yes_no, long_text), and two hidden fields (`customer_id`, `project_type`).
  - Timeout: 10 000 ms.
- **Continue on Fail:** Enabled.
- **Input:** Filter Ready for Follow-up.
- **Output:** Not connected to any downstream node directly — its output is referenced by the Send Feedback Request Email node via `$('Generate Personalized Feedback Form').item.json.form_url`.
- **Edge Cases:** If Typeform API fails, `form_url` will be undefined and the email will fall back to the hardcoded `https://form.example.com/feedback`. The `YOUR_TOKEN_HERE` placeholder must be replaced. Typeform API may require additional account-level configuration (workspace, theme).

---

**Send Feedback Request Email**
- **Type:** `n8n-nodes-base.httpRequest` (v4.2)
- **Technical Role:** Sends a personalized review-request email through the SendGrid API using a dynamic template.
- **Configuration:**
  - URL: `https://api.sendgrid.com/v3/mail/send`
  - Method: POST
  - Headers: `Authorization: Bearer YOUR_TOKEN_HERE`, `Content-Type: application/json`
  - JSON body (dynamic): Includes `personalizations` with `to` (email + name), `dynamic_template_data` (customer_name, project_type, completion_date, feedback_form_url, company_name), `from` (reviews@yourcompany.com), `template_id` (`YOUR_SENDGRID_TEMPLATE_ID`).
  - Timeout: 10 000 ms.
- **Continue on Fail:** Enabled.
- **Input:** Filter Ready for Follow-up.
- **Output:** None (terminal node for Phase A).
- **Edge Cases:** SendGrid 401 if API key is invalid; 400 if `template_id` does not exist or `dynamic_template_data` fields don't match template variables. The `feedback_form_url` references the Typeform node — if that node failed, it falls back to the hardcoded URL. The placeholder values must all be replaced with real credentials and template IDs.

---

#### Block 3 — AI Processing & Enhancement (Claude)

**Overview:** This block receives the customer's submitted feedback via a second webhook, parses it, and passes it through two Claude AI agents in parallel — one for sentiment/quality analysis, one for testimonial generation. Both agents share a single Claude model node. Their outputs are merged and parsed into a unified data structure.

**Nodes Involved:**
- Webhook - Form Submission Received
- Parse Feedback Response
- AI Sentiment & Quality Analysis
- Generate Polished Testimonial with Claude
- Claude AI Model
- Parse AI Generated Content

---

**Webhook - Form Submission Received**
- **Type:** `n8n-nodes-base.webhook` (v1.1)
- **Technical Role:** HTTP POST entry point for receiving completed feedback form submissions (e.g., Typeform webhook callback).
- **Configuration:** Path `feedback-submission`, method `POST`, response mode `responseNode`.
- **Input:** External HTTP POST from Typeform or similar form service.
- **Output:** Connects to **Parse Feedback Response**.
- **Edge Cases:** The expected payload shape is `form_response` with `answers` array and `hidden` fields. If a different form platform is used, the parsing logic must be adapted.

---

**Parse Feedback Response**
- **Type:** `n8n-nodes-base.code` (v2)
- **Technical Role:** Custom JavaScript that extracts structured fields from the raw form submission.
- **Configuration:** Mode `runOnceForEachItem`.
- **Key Logic:**
  - Extracts `rating` from `opinion_scale` answer (defaults to 5)
  - Extracts `whatLiked` from answer whose field title contains "like most"
  - Extracts `results` from answer whose field title contains "results"
  - Extracts `wouldRecommend` from `yes_no` answer (defaults to `true`)
  - Extracts `additionalFeedback` from answer whose field title contains "additional"
  - Extracts hidden fields: `customer_id`, `project_type`
  - Concatenates non-empty feedback parts into `fullText`
- **Output Structure:** `submissionId`, `customerId`, `projectType`, `rating`, `wouldRecommend`, `feedback` (whatLiked, results, additionalFeedback, fullText), `metadata` (submittedAt, formId, responseId), `rawSubmission`.
- **Input:** Webhook - Form Submission Received.
- **Output:** Connects to **AI Sentiment & Quality Analysis** and **Generate Polished Testimonial with Claude** (parallel).
- **Edge Cases:** If the answer array is empty or uses different field types, extraction falls back to defaults (rating=5, wouldRecommend=true, empty strings). If `hidden` fields are missing, `customerId` defaults to `'UNKNOWN'`.

---

**AI Sentiment & Quality Analysis**
- **Type:** `@n8n/n8n-nodes-langchain.agent` (v1.6)
- **Technical Role:** LangChain Agent node that sends a prompt to the connected Claude model requesting structured sentiment/quality analysis of the customer feedback.
- **Configuration:**
  - Prompt type: `define` (inline text prompt)
  - Prompt text (dynamic): Includes rating, wouldRecommend, whatLiked, results, additionalFeedback. Requests JSON output with: sentiment, sentimentScore, qualityScore, keyThemes, benefitsMentioned, emotionalTone, bestQuotes, improvementSuggestions, authenticityScore, isTestimonialWorthy, reasoning.
  - System message: *"You are an expert in analyzing customer feedback and testimonials. Return only valid JSON with no markdown formatting."*
- **Connected AI Model:** Claude AI Model (via `ai_languageModel` connection).
- **Input:** Parse Feedback Response.
- **Output:** Connects to **Parse AI Generated Content**.
- **Edge Cases:** If Claude returns markdown-wrapped JSON (````json ... ````), the downstream Code node strips those markers. If the response is not valid JSON, a fallback object is used. Token limits on very long feedback could truncate the response.

---

**Generate Polished Testimonial with Claude**
- **Type:** `@n8n/n8n-nodes-langchain.agent` (v1.6)
- **Technical Role:** LangChain Agent node that generates multiple polished testimonial versions from the raw feedback, informed by the sentiment analysis.
- **Configuration:**
  - Prompt type: `define` (inline text prompt)
  - Prompt text (dynamic): Includes rating, customerId, projectType, fullText, and the full sentiment analysis JSON (pulled from the AI Sentiment & Quality Analysis node via `$('AI Sentiment & Quality Analysis').item.json`). Requests JSON output with: testimonials (short/medium/long with text, headline, bestFor), pullQuote, socialMediaPost, metaDescription, keyBenefits, recommendedPlatforms, contentRating.
  - System message: *"You are an expert copywriter specializing in testimonials. Transform raw feedback into compelling, authentic testimonials that maintain the customer's voice. Return only valid JSON."*
- **Connected AI Model:** Claude AI Model (via `ai_languageModel` connection).
- **Input:** Parse Feedback Response.
- **Output:** Connects to **Parse AI Generated Content**.
- **Edge Cases:** Same JSON-parsing fallback applies. If the sentiment analysis node hasn't completed yet (race condition unlikely since both run from the same input), the reference to that node may fail. The `temperature` of 0.3 (set on the model node) reduces randomness but does not eliminate it — occasional non-JSON output is still possible.

---

**Claude AI Model**
- **Type:** `@n8n/n8n-nodes-langchain.lmChatAnthropic` (v1)
- **Technical Role:** Shared Anthropic Claude language model node used by both AI Agent nodes.
- **Configuration:**
  - Model: `claude-sonnet-4-20250514`
  - Temperature: 0.3
  - Credentials: Anthropic API (credential placeholder: `Anthropic account - test`)
- **Connections:** `ai_languageModel` output connects to both **AI Sentiment & Quality Analysis** and **Generate Polished Testimonial with Claude**.
- **Edge Cases:** Invalid or expired API key causes both agents to fail. The model name must be available on the connected Anthropic account. Rate limiting on high-volume usage. The `claude-sonnet-4-20250514` model identifier must match an actual model available in the Anthropic account.

---

**Parse AI Generated Content**
- **Type:** `n8n-nodes-base.code` (v2)
- **Technical Role:** Merges outputs from both AI agents, parses their JSON responses (with fallback), and assembles a unified item combining original feedback, sentiment analysis, and generated testimonials.
- **Configuration:** Mode `runOnceForEachItem`.
- **Key Logic:**
  - Reads `AI Sentiment & Quality Analysis` output via `$('AI Sentiment & Quality Analysis').item.json`
  - Reads `Generate Polished Testimonial with Claude` output via `$('Generate Polished Testimonial with Claude').item.json`
  - Reads original parsed feedback via `$('Parse Feedback Response').item.json`
  - Strips ````json` and ```` markdown fences from both AI responses
  - Attempts `JSON.parse`; on failure, provides a fallback object (sentiment defaults to POSITIVE/qualityScore 70/isTestimonialWorthy true; testimonial defaults to first 100 chars of original feedback)
- **Output Structure:** `submissionId`, `customerId`, `projectType`, `rating`, `wouldRecommend`, `originalFeedback`, `sentimentAnalysis`, `generatedTestimonials`, `metadata` (with `processedAt` appended).
- **Input:** AI Sentiment & Quality Analysis and Generate Polished Testimonial with Claude (both feed into this node).
- **Output:** Connects to **Filter High-Quality Testimonials** and **Log to Analytics Dashboard** (parallel).
- **Edge Cases:** If either AI agent times out or returns a non-parseable response, the fallback objects ensure the workflow does not break. However, the fallback sentiment defaults to `isTestimonialWorthy: true`, which could let low-quality feedback through if both AIs fail.

---

#### Block 4 — Publish, Distribute & Track

**Overview:** This block filters for high-quality testimonials, formats them for multiple publishing platforms, distributes them to WordPress, Trustpilot, and the CRM, logs everything to an Airtable analytics dashboard, sends a thank-you email, and returns a success response to the feedback-submission webhook caller.

**Nodes Involved:**
- Filter High-Quality Testimonials
- Format for Publishing Platforms
- Publish to Website (WordPress)
- Submit to Trustpilot
- Update CRM with Testimonial Status
- Log to Analytics Dashboard
- Send Thank You Email
- Return Success Response

---

**Filter High-Quality Testimonials**
- **Type:** `n8n-nodes-base.filter` (v2.2)
- **Technical Role:** Passes only testimonials that meet quality thresholds — minimum 4-star rating, `isTestimonialWorthy` is true, and quality score ≥ 60.
- **Configuration:** Combinator `and`, three conditions:
  1. `$json.rating >= 4` (number gte)
  2. `$json.sentimentAnalysis.isTestimonialWorthy` is `true` (boolean)
  3. `$json.sentimentAnalysis.qualityScore >= 60` (number gte)
- **Input:** Parse AI Generated Content.
- **Output:** Connects to **Format for Publishing Platforms**.
- **Edge Cases:** Items that don't pass are silently dropped. If the AI fallback was used (qualityScore defaults to 70), they will still pass — this is intentional but may allow unreviewed content through. If rating is a string (e.g., "4"), the strict type validation may reject it.

---

**Format for Publishing Platforms**
- **Type:** `n8n-nodes-base.code` (v2)
- **Technical Role:** Generates platform-specific representations of each testimonial — HTML for website, JSON for Google, structured payloads for Trustpilot, social media variants, and an email signature quote.
- **Configuration:** Runs in default mode (`runOnceForAllItems`).
- **Key Logic:**
  - Generates a `platformFormats` object containing: `website` (html + json), `google` (rating + comment), `trustpilot` (rating + title + content), `socialMedia` (twitter, linkedin, instagram), `emailSignature` (text).
  - HTML includes star rating characters, blockquote, cite, and project type.
  - Sets `publishReady: true`.
- **Input:** Filter High-Quality Testimonials.
- **Output:** Connects to **Publish to Website (WordPress)**, **Submit to Trustpilot**, and **Update CRM with Testimonial Status** (all three in parallel).
- **Edge Cases:** If `generatedTestimonials.testimonials.medium` is undefined (e.g., AI returned the fallback), accessing `.text` or `.headline` will throw — the code assumes a well-structured `generatedTestimonials` object. Star characters (★/☆) are Unicode and may need charset support on the target platform.

---

**Publish to Website (WordPress)**
- **Type:** `n8n-nodes-base.httpRequest` (v4.2)
- **Technical Role:** Creates a new "testimonial" post in WordPress via the REST API.
- **Configuration:**
  - URL: `https://yoursite.com/wp-json/wp/v2/testimonials`
  - Method: POST
  - Headers: `Authorization: Basic YOUR_WORDPRESS_AUTH_TOKEN`, `Content-Type: application/json`
  - JSON body (dynamic): `title` (medium headline), `content` (website HTML), `status: "publish"`, `meta` (rating, customer_id, project_type, submission_date).
  - Timeout: 10 000 ms.
- **Continue on Fail:** Enabled.
- **Input:** Format for Publishing Platforms.
- **Output:** Connects to **Send Thank You Email**.
- **Edge Cases:** WordPress endpoint `/wp/v2/testimonials` requires a custom post type named "testimonials" registered on the site. Basic auth must be enabled (application password). 401/403 if credentials are wrong. The placeholder URL must be replaced.

---

**Submit to Trustpilot**
- **Type:** `n8n-nodes-base.httpRequest` (v4.2)
- **Technical Role:** Submits the review to Trustpilot's private API.
- **Configuration:**
  - URL: `https://api.trustpilot.com/v1/private/business-units/YOUR_BUSINESS_ID/reviews`
  - Method: POST
  - Headers: `Authorization: Bearer YOUR_TOKEN_HERE`, `Content-Type: application/json`
  - JSON body (dynamic): `consumer` (email from Check Optimal Follow-up Timing node, name), `rating`, `title` (medium headline), `text` (trustpilot content), `referenceId`.
  - Timeout: 10 000 ms.
- **Continue on Fail:** Enabled.
- **Input:** Format for Publishing Platforms.
- **Output:** Connects to **Send Thank You Email**.
- **Edge Cases:** Trustpilot's private API requires specific permissions and a verified business unit. The `YOUR_BUSINESS_ID` placeholder must be replaced. Customer email/name is pulled from the Check Optimal Follow-up Timing node, which may not be in the current execution context if the workflow was triggered only via the feedback-submission webhook (Phase B only). This cross-reference could return stale or missing data.

---

**Update CRM with Testimonial Status**
- **Type:** `n8n-nodes-base.httpRequest` (v4.2)
- **Technical Role:** Patches the HubSpot contact record to mark the testimonial as submitted and store rating, date, text, and quality score.
- **Configuration:**
  - URL: `https://api.hubspot.com/crm/v3/objects/contacts/{{ $json.customerId }}`
  - Method: PATCH
  - Headers: `Authorization: Bearer YOUR_TOKEN_HERE`, `Content-Type: application/json`
  - JSON body (dynamic): Properties `testimonial_submitted: "true"`, `testimonial_rating`, `testimonial_date`, `testimonial_text` (short version), `last_review_score` (quality score).
  - Timeout: 10 000 ms.
- **Continue on Fail:** Enabled.
- **Input:** Format for Publishing Platforms.
- **Output:** Connects to **Send Thank You Email**.
- **Edge Cases:** If `customerId` is "UNKNOWN" (from the feedback parser fallback), the PATCH will fail with 404. HubSpot properties (`testimonial_submitted`, `testimonial_rating`, etc.) must exist in the HubSpot account or the API will reject them. Token placeholder must be replaced.

---

**Log to Analytics Dashboard**
- **Type:** `n8n-nodes-base.httpRequest` (v4.2)
- **Technical Role:** Creates a record in Airtable for analytics and reporting.
- **Configuration:**
  - URL: `https://api.airtable.com/v0/YOUR_BASE_ID/Testimonials`
  - Method: POST
  - Headers: `Authorization: Bearer YOUR_TOKEN_HERE`, `Content-Type: application/json`
  - JSON body (dynamic): Fields include Submission ID, Customer ID, Rating, Sentiment, Quality Score, Published, Submitted At, Project Type, Short Version, Medium Version.
  - Timeout: 10 000 ms.
- **Continue on Fail:** Enabled.
- **Input:** Parse AI Generated Content (bypasses the quality filter — logs all submissions).
- **Output:** None (terminal logging node).
- **Edge Cases:** Airtable base and table ("Testimonials") must exist, and field names must match exactly. The `YOUR_BASE_ID` and token must be configured. This node receives items regardless of whether they pass the quality filter, so it logs both publishable and non-publishable submissions.

---

**Send Thank You Email**
- **Type:** `n8n-nodes-base.httpRequest` (v4.2)
- **Technical Role:** Sends an automated appreciation email to the customer.
- **Configuration:**
  - URL: `https://api.sendgrid.com/v3/mail/send`
  - Method: POST
  - Headers: `Authorization: Bearer YOUR_TOKEN_HERE`, `Content-Type: application/json`
  - JSON body (dynamic): `personalizations` with `to` (email from Check Optimal Follow-up Timing node), `dynamic_template_data` (customer_name, testimonial_used: true), `from` (thanks@yourcompany.com), `template_id` (`YOUR_THANKYOU_TEMPLATE_ID`).
  - Timeout: 10 000 ms.
- **Continue on Fail:** Not explicitly enabled (defaults to false).
- **Input:** Receives items from **Publish to Website (WordPress)**, **Submit to Trustpilot**, or **Update CRM with Testimonial Status** (three parallel inputs).
- **Output:** Connects to **Return Success Response**.
- **Edge Cases:** Because three nodes feed into this one, the same item may arrive multiple times. In practice, since the three input branches are parallel outputs of the same Format for Publishing Platforms item, n8n's v1 execution order will process them in sequence, potentially sending up to three thank-you emails for the same testimonial. The email address is referenced from the Check Optimal Follow-up Timing node, which may not be available in Phase B–only executions. The template ID placeholder must be replaced.

---

**Return Success Response**
- **Type:** `n8n-nodes-base.respondToWebhook` (v1)
- **Technical Role:** Sends a JSON response back to the caller of the feedback-submission webhook.
- **Configuration:**
  - Respond with: `json`
  - Response body (expression): `{ success: true, message: 'Testimonial processed and published', submissionId: $json.submissionId }`
  - Response headers: `Content-Type: application/json`
- **Input:** Send Thank You Email.
- **Output:** None (terminal response node).
- **Edge Cases:** This node only responds to the **Webhook - Form Submission Received** entry point. The **Webhook Trigger - Project Completion** also has `responseNode` mode but has no connected Respond to Webhook node — it will hang and eventually time out without a response. Consider adding a response node for that webhook as well.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Webhook Trigger - Project Completion | webhook (v1.1) | HTTP POST entry for project/purchase completion events | External caller | Prepare Customer Context | 1. Trigger & Customer Data |
| Daily Batch Check | scheduleTrigger (v1.2) | Cron trigger at 09:00 UTC for batch processing | — (time-based) | Prepare Customer Context | 1. Trigger & Customer Data |
| Prepare Customer Context | set (v3.4) | Normalizes incoming payload into unified schema | Webhook Trigger - Project Completion, Daily Batch Check | Fetch Full Customer Profile from CRM | 1. Trigger & Customer Data |
| Fetch Full Customer Profile from CRM | httpRequest (v4.2) | GET customer record from HubSpot CRM | Prepare Customer Context | Check Optimal Follow-up Timing | 2. Smart Follow-up & Feedback Collection |
| Check Optimal Follow-up Timing | code (v2) | Calculates follow-up window and enriches item with timing/customer/project data | Fetch Full Customer Profile from CRM | Filter Ready for Follow-up | 2. Smart Follow-up & Feedback Collection |
| Filter Ready for Follow-up | filter (v2.2) | Passes only items ready for follow-up with valid email | Check Optimal Follow-up Timing | Generate Personalized Feedback Form, Send Feedback Request Email | 2. Smart Follow-up & Feedback Collection |
| Generate Personalized Feedback Form | httpRequest (v4.2) | Creates a personalized Typeform feedback form | Filter Ready for Follow-up | — (referenced by Send Feedback Request Email) | 2. Smart Follow-up & Feedback Collection |
| Send Feedback Request Email | httpRequest (v4.2) | Sends review-request email via SendGrid dynamic template | Filter Ready for Follow-up | — | 2. Smart Follow-up & Feedback Collection |
| Webhook - Form Submission Received | webhook (v1.1) | HTTP POST entry for feedback form submissions | External caller (Typeform callback) | Parse Feedback Response | 3. AI Processing & Enhancement (Claude) |
| Parse Feedback Response | code (v2) | Extracts structured fields from raw form submission | Webhook - Form Submission Received | AI Sentiment & Quality Analysis, Generate Polished Testimonial with Claude | 3. AI Processing & Enhancement (Claude) |
| AI Sentiment & Quality Analysis | @n8n/n8n-nodes-langchain.agent (v1.6) | Claude-powered sentiment/quality analysis of feedback | Parse Feedback Response | Parse AI Generated Content | 3. AI Processing & Enhancement (Claude) |
| Generate Polished Testimonial with Claude | @n8n/n8n-nodes-langchain.agent (v1.6) | Claude-powered testimonial generation from raw feedback | Parse Feedback Response | Parse AI Generated Content | 3. AI Processing & Enhancement (Claude) |
| Claude AI Model | @n8n/n8n-nodes-langchain.lmChatAnthropic (v1) | Shared Anthropic Claude model (claude-sonnet-4-20250514, temp 0.3) | — | AI Sentiment & Quality Analysis (ai_languageModel), Generate Polished Testimonial with Claude (ai_languageModel) | 3. AI Processing & Enhancement (Claude) |
| Parse AI Generated Content | code (v2) | Merges and parses both AI outputs with fallback into unified structure | AI Sentiment & Quality Analysis, Generate Polished Testimonial with Claude | Filter High-Quality Testimonials, Log to Analytics Dashboard | 4. Publish, Distribute & Track |
| Filter High-Quality Testimonials | filter (v2.2) | Passes only testimonials with rating ≥ 4, isTestimonialWorthy, qualityScore ≥ 60 | Parse AI Generated Content | Format for Publishing Platforms | 4. Publish, Distribute & Track |
| Format for Publishing Platforms | code (v2) | Generates platform-specific testimonial formats (HTML, Google, Trustpilot, social, email signature) | Filter High-Quality Testimonials | Publish to Website (WordPress), Submit to Trustpilot, Update CRM with Testimonial Status | 4. Publish, Distribute & Track |
| Publish to Website (WordPress) | httpRequest (v4.2) | Creates a published testimonial post in WordPress REST API | Format for Publishing Platforms | Send Thank You Email | 4. Publish, Distribute & Track |
| Submit to Trustpilot | httpRequest (v4.2) | Submits review to Trustpilot private API | Format for Publishing Platforms | Send Thank You Email | 4. Publish, Distribute & Track |
| Update CRM with Testimonial Status | httpRequest (v4.2) | Patches HubSpot contact with testimonial status and details | Format for Publishing Platforms | Send Thank You Email | 4. Publish, Distribute & Track |
| Log to Analytics Dashboard | httpRequest (v4.2) | Logs submission data to Airtable analytics base | Parse AI Generated Content | — | 4. Publish, Distribute & Track |
| Send Thank You Email | httpRequest (v4.2) | Sends automated thank-you email via SendGrid | Publish to Website (WordPress), Submit to Trustpilot, Update CRM with Testimonial Status | Return Success Response | 4. Publish, Distribute & Track |
| Return Success Response | respondToWebhook (v1) | Returns JSON success response to feedback-submission webhook caller | Send Thank You Email | — | 4. Publish, Distribute & Track |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n named "Automated Testimonial & Review Collector".

2. **Add a Sticky Note** for documentation. Set content to the main overview text (the large sticky note content from the original workflow). Position it at the far left of the canvas.

3. **Create the first entry point — Webhook Trigger - Project Completion**
   - Add a **Webhook** node.
   - Set HTTP Method to `POST`.
   - Set Path to `testimonial-trigger`.
   - Set Response Mode to `Respond to Webhook` (using a response node later).
   - Rename to "Webhook Trigger - Project Completion".

4. **Create the second entry point — Daily Batch Check**
   - Add a **Schedule Trigger** node.
   - Set the cron expression to `0 9 * * *` (daily at 09:00 UTC).
   - Rename to "Daily Batch Check".

5. **Create Prepare Customer Context**
   - Add a **Set** node.
   - Add 8 assignments with these names, types, and expressions:
     - `customerId` (string): `={{ $json.body?.customerId || $json.customerId || 'MANUAL-' + Date.now() }}`
     - `customerEmail` (string): `={{ $json.body?.customerEmail || $json.customerEmail }}`
     - `customerName` (string): `={{ $json.body?.customerName || $json.customerName }}`
     - `projectType` (string): `={{ $json.body?.projectType || $json.projectType || 'General Service' }}`
     - `completionDate` (string): `={{ $json.body?.completionDate || $json.completionDate || new Date().toISOString() }}`
     - `followUpDelay` (number): `={{ $json.body?.followUpDelay || 7 }}`
     - `preferredChannel` (string): `={{ $json.body?.preferredChannel || 'email' }}`
     - `jobId` (string): `=REVIEW-{{ Date.now() }}-{{ Math.random().toString(36).substr(2, 6).toUpperCase() }}`
   - Connect both **Webhook Trigger - Project Completion** and **Daily Batch Check** → **Prepare Customer Context**.

6. **Create Fetch Full Customer Profile from CRM**
   - Add an **HTTP Request** node.
   - Method: `GET`.
   - URL: `=https://api.hubspot.com/crm/v3/objects/contacts/{{ $json.customerId }}`
   - Enable **Send Headers**. Add `Authorization: Bearer YOUR_HUBSPOT_TOKEN` and `Content-Type: application/json`.
   - Timeout: 10 000 ms.
   - Enable **Continue On Fail**.
   - Connect **Prepare Customer Context** → **Fetch Full Customer Profile from CRM**.

7. **Create Check Optimal Follow-up Timing**
   - Add a **Code** node.
   - Mode: `Run Once for Each Item`.
   - Paste the full JavaScript from the original (calculates `daysSinceCompletion`, `isReadyForFollowup`, `shouldSendFollowup`, and builds `customerData`, `timing`, `projectContext` objects by reading CRM properties).
   - Connect **Fetch Full Customer Profile from CRM** → **Check Optimal Follow-up Timing**.

8. **Create Filter Ready for Follow-up**
   - Add a **Filter** node.
   - Combinator: `and`.
   - Condition 1: `$json.timing.shouldSendFollowup` is `true` (boolean).
   - Condition 2: `$json.customerData.email` `exists` (string).
   - Connect **Check Optimal Follow-up Timing** → **Filter Ready for Follow-up**.

9. **Create Generate Personalized Feedback Form**
   - Add an **HTTP Request** node.
   - Method: `POST`. URL: `https://api.typeform.com/forms`.
   - Enable **Send Headers**: `Authorization: Bearer YOUR_TYPEFORM_TOKEN`, `Content-Type: application/json`.
   - Specify Body: `JSON`. Paste the JSON body template with dynamic interpolation for project type and hidden fields (`customer_id`, `project_type`).
   - Timeout: 10 000 ms. Enable **Continue On Fail**.
   - Connect **Filter Ready for Follow-up** → **Generate Personalized Feedback Form**.

10. **Create Send Feedback Request Email**
    - Add an **HTTP Request** node.
    - Method: `POST`. URL: `https://api.sendgrid.com/v3/mail/send`.
    - Enable **Send Headers**: `Authorization: Bearer YOUR_SENDGRID_TOKEN`, `Content-Type: application/json`.
    - Specify Body: `JSON`. Paste the JSON body template referencing `customerData.email`, `customerData.name`, `projectContext.type`, `timing.completionDate`, and the Typeform node's `form_url` (with fallback to `https://form.example.com/feedback`).
    - Set `from` email to your review address and `template_id` to your SendGrid dynamic template ID.
    - Timeout: 10 000 ms. Enable **Continue On Fail**.
    - Connect **Filter Ready for Follow-up** → **Send Feedback Request Email**.

11. **Create Webhook - Form Submission Received**
    - Add a **Webhook** node.
    - HTTP Method: `POST`. Path: `feedback-submission`.
    - Response Mode: `Respond to Webhook`.
    - This is the second entry point.

12. **Create Parse Feedback Response**
    - Add a **Code** node.
    - Mode: `Run Once for Each Item`.
    - Paste the JavaScript that extracts `rating`, `whatLiked`, `results`, `wouldRecommend`, `additionalFeedback`, hidden fields, and concatenates into `fullText`.
    - Connect **Webhook - Form Submission Received** → **Parse Feedback Response**.

13. **Create the Claude AI Model node**
    - Add an **Anthropic Chat Model** node (`@n8n/n8n-nodes-langchain.lmChatAnthropic`).
    - Model: `claude-sonnet-4-20250514`.
    - Temperature: 0.3.
    - Configure **Anthropic API credentials** with a valid API key.

14. **Create AI Sentiment & Quality Analysis**
    - Add an **AI Agent** node (`@n8n/n8n-nodes-langchain.agent`).
    - Prompt Type: `Define`.
    - Paste the sentiment analysis prompt text that includes the feedback fields and requests the structured JSON output.
    - System Message: *"You are an expert in analyzing customer feedback and testimonials. Return only valid JSON with no markdown formatting."*
    - Connect **Parse Feedback Response** → **AI Sentiment & Quality Analysis** (main input).
    - Connect **Claude AI Model** → **AI Sentiment & Quality Analysis** (ai_languageModel connection).

15. **Create Generate Polished Testimonial with Claude**
    - Add an **AI Agent** node.
    - Prompt Type: `Define`.
    - Paste the testimonial generation prompt that includes the feedback, references the sentiment analysis node output, and requests the multi-version testimonial JSON.
    - System Message: *"You are an expert copywriter specializing in testimonials. Transform raw feedback into compelling, authentic testimonials that maintain the customer's voice. Return only valid JSON."*
    - Connect **Parse Feedback Response** → **Generate Polished Testimonial with Claude** (main input).
    - Connect **Claude AI Model** → **Generate Polished Testimonial with Claude** (ai_languageModel connection).

16. **Create Parse AI Generated Content**
    - Add a **Code** node.
    - Mode: `Run Once for Each Item`.
    - Paste the JavaScript that reads both AI agent outputs, strips markdown fences, attempts JSON parse with fallback, and assembles the unified output with `submissionId`, `customerId`, `projectType`, `rating`, `wouldRecommend`, `originalFeedback`, `sentimentAnalysis`, `generatedTestimonials`, `metadata`.
    - Connect **AI Sentiment & Quality Analysis** → **Parse AI Generated Content** and **Generate Polished Testimonial with Claude** → **Parse AI Generated Content**.

17. **Create Filter High-Quality Testimonials**
    - Add a **Filter** node.
    - Combinator: `and`.
    - Condition 1: `$json.rating >= 4` (number).
    - Condition 2: `$json.sentimentAnalysis.isTestimonialWorthy` is `true` (boolean).
    - Condition 3: `$json.sentimentAnalysis.qualityScore >= 60` (number).
    - Connect **Parse AI Generated Content** → **Filter High-Quality Testimonials**.

18. **Create Log to Analytics Dashboard**
    - Add an **HTTP Request** node.
    - Method: `POST`. URL: `https://api.airtable.com/v0/YOUR_BASE_ID/Testimonials`.
    - Headers: `Authorization: Bearer YOUR_AIRTABLE_TOKEN`, `Content-Type: application/json`.
    - Specify Body: `JSON`. Include fields for Submission ID, Customer ID, Rating, Sentiment, Quality Score, Published, Submitted At, Project Type, Short Version, Medium Version.
    - Timeout: 10 000 ms. Enable **Continue On Fail**.
    - Connect **Parse AI Generated Content** → **Log to Analytics Dashboard** (this branch bypasses the quality filter).

19. **Create Format for Publishing Platforms**
    - Add a **Code** node.
    - Paste the JavaScript that builds `platformFormats` (website HTML/json, google, trustpilot, socialMedia variants, emailSignature) and sets `publishReady: true`.
    - Connect **Filter High-Quality Testimonials** → **Format for Publishing Platforms**.

20. **Create Publish to Website (WordPress)**
    - Add an **HTTP Request** node.
    - Method: `POST`. URL: `https://yoursite.com/wp-json/wp/v2/testimonials`.
    - Headers: `Authorization: Basic YOUR_WORDPRESS_AUTH_TOKEN`, `Content-Type: application/json`.
    - JSON body: `title`, `content` (HTML from platformFormats), `status: "publish"`, `meta` (rating, customer_id, project_type, submission_date).
    - Timeout: 10 000 ms. Enable **Continue On Fail**.
    - Connect **Format for Publishing Platforms** → **Publish to Website (WordPress)**.

21. **Create Submit to Trustpilot**
    - Add an **HTTP Request** node.
    - Method: `POST`. URL: `https://api.trustpilot.com/v1/private/business-units/YOUR_BUSINESS_ID/reviews`.
    - Headers: `Authorization: Bearer YOUR_TRUSTPILOT_TOKEN`, `Content-Type: application/json`.
    - JSON body: `consumer` (email + name from Check Optimal Follow-up Timing node), `rating`, `title`, `text`, `referenceId`.
    - Timeout: 10 000 ms. Enable **Continue On Fail**.
    - Connect **Format for Publishing Platforms** → **Submit to Trustpilot**.

22. **Create Update CRM with Testimonial Status**
    - Add an **HTTP Request** node.
    - Method: `PATCH`. URL: `=https://api.hubspot.com/crm/v3/objects/contacts/{{ $json.customerId }}`.
    - Headers: `Authorization: Bearer YOUR_HUBSPOT_TOKEN`, `Content-Type: application/json`.
    - JSON body: Properties `testimonial_submitted: "true"`, `testimonial_rating`, `testimonial_date`, `testimonial_text` (short version), `last_review_score`.
    - Timeout: 10 000 ms. Enable **Continue On Fail**.
    - Connect **Format for Publishing Platforms** → **Update CRM with Testimonial Status**.

23. **Create Send Thank You Email**
    - Add an **HTTP Request** node.
    - Method: `POST`. URL: `https://api.sendgrid.com/v3/mail/send`.
    - Headers: `Authorization: Bearer YOUR_SENDGRID_TOKEN`, `Content-Type: application/json`.
    - JSON body: `personalizations` (to email from Check Optimal Follow-up Timing node, dynamic_template_data with customer_name and testimonial_used), `from` (thanks@yourcompany.com), `template_id`.
    - Timeout: 10 000 ms.
    - Connect **Publish to Website (WordPress)** → **Send Thank You Email**.
    - Connect **Submit to Trustpilot** → **Send Thank You Email**.
    - Connect **Update CRM with Testimonial Status** → **Send Thank You Email**.

24. **Create Return Success Response**
    - Add a **Respond to Webhook** node.
    - Respond With: `JSON`.
    - Response Body: `={{ JSON.stringify({ success: true, message: 'Testimonial processed and published', submissionId: $json.submissionId }, null, 2) }}`.
    - Add Response Header: `Content-Type: application/json`.
    - Connect **Send Thank You Email** → **Return Success Response**.

25. **Add section Sticky Notes** (optional but recommended):
    - Sticky Note1 (blue, "1. Trigger & Customer Data") covering the first three nodes.
    - Sticky Note2 (blue, "2. Smart Follow-up & Feedback Collection") covering nodes from Fetch CRM through Send Email.
    - Sticky Note3 (green, "3. AI Processing & Enhancement (Claude)") covering the feedback webhook through Claude model.
    - Sticky Note4 (green, "4. Publish, Distribute & Track") covering from Parse AI through Return Success.

26. **Configure all credentials** before activating:
    - Anthropic API key for Claude model node.
    - HubSpot Private App Token (replacing all `YOUR_TOKEN_HERE` for HubSpot calls).
    - SendGrid API Key (replacing tokens for feedback request and thank-you emails).
    - Typeform Personal Access Token (for form creation).
    - Trustpilot API Key and Business Unit ID.
    - WordPress Application Password (Base64-encoded for Basic auth).
    - Airtable Personal Access Token and Base ID.
    - Replace all template IDs (`YOUR_SENDGRID_TEMPLATE_ID`, `YOUR_THANKYOU_TEMPLATE_ID`).
    - Replace all URL placeholders (`yoursite.com`, `YOUR_BUSINESS_ID`, `YOUR_BASE_ID`).

27. **Activate the workflow.** Test by sending a POST request to the `/testimonial-trigger` webhook with the sample payload, then simulate a form submission to `/feedback-submission`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| The main overview sticky note contains an extensive feature list including multi-touch follow-up, A/B testing of messaging, video testimonial collection, and auto-detection of negative feedback — these are described as planned features but are **not implemented** in the current workflow JSON. | Main Sticky Note |
| The **Webhook Trigger - Project Completion** node uses `responseNode` mode but no Respond to Webhook node is connected to it. The caller will time out without a response. A response node should be added or the mode should be changed to `lastNode`. | Architecture concern |
| The **Send Thank You Email** node receives input from three parallel branches (WordPress, Trustpilot, CRM update). In n8n v1 execution order, this can result in duplicate thank-you emails being sent for the same testimonial. Consider using a Merge node or a No-Op/Wait pattern to deduplicate. | Architecture concern |
| Cross-node references (e.g., `$('Check Optimal Follow-up Timing')` in Trustpilot and Thank You Email nodes) depend on data from Phase A. If the workflow is triggered solely via the feedback-submission webhook (Phase B), these references may return stale or missing data. | Data flow concern |
| All HTTP Request nodes use hardcoded placeholder tokens (`YOUR_TOKEN_HERE`). These must be replaced with actual credentials or, better, with n8n credential objects before activation. | Security / setup |
| The Daily Batch Check trigger fires but does not fetch a list of eligible customers from the CRM. It relies on external data injection. For full batch automation, add an HTTP Request node querying HubSpot for contacts where `testimonial_submitted` is not `true` and whose project completion date falls within the follow-up window. | Feature gap |
| Claude model identifier `claude-sonnet-4-20250514` must be available on the connected Anthropic account. Verify model availability at the Anthropic console before activation. | Model compatibility |
| The Parse AI Generated Content Code node's fallback sets `isTestimonialWorthy: true` and `qualityScore: 70` when AI parsing fails, which may let unreviewed content through the quality filter. Consider setting fallback values to `false` / `0` for safer behavior. | Reliability concern |