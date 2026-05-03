Personalize Cold Emails with LinkedIn Data, GPT-4 Analysis & Airtable-Instantly Integration

https://n8nworkflows.xyz/workflows/personalize-cold-emails-with-linkedin-data--gpt-4-analysis---airtable-instantly-integration-9243


# Personalize Cold Emails with LinkedIn Data, GPT-4 Analysis & Airtable-Instantly Integration

### 1. Workflow Overview

This workflow, titled **"Personalize Cold Emails with LinkedIn Data, GPT-4 Analysis & Airtable-Instantly Integration"**, automates the enrichment of sales leads by combining data from Airtable, LinkedIn company and personal profiles, company websites, and LinkedIn posts. It leverages advanced AI analysis (GPT-4) to generate personalized insights and cold email content, which is then uploaded to the Instantly outreach platform.

The workflow is designed primarily for sales and marketing teams aiming to personalize outreach campaigns at scale by blending company research, decision-maker profiles, and recent activity with AI-generated messaging.

**Logical blocks and their roles:**

- **1.1 Input Reception & Lead Retrieval:** Receives input via webhook and fetches lead data from Airtable.
- **1.2 Email Validation:** Validates lead email addresses using NeverBounce.
- **1.3 Geographic & Business Model Filtering:** Filters leads based on location (US) and business model (B2B).
- **1.4 Website & Company Data Scraping:** Scrapes company website and LinkedIn company profiles.
- **1.5 HTML Cleaning & Text Extraction:** Removes HTML from scraped pages to extract clean text.
- **1.6 AI Analysis of Company Data:** Uses GPT-4 to analyze company mission, offerings, process, and proof of success.
- **1.7 LinkedIn Profile Scraping & Analysis:** Scrapes and analyzes LinkedIn personal profiles and posts.
- **1.8 Personalized Email Content Generation:** Extracts variables and generates personalized cold email copy.
- **1.9 Data Cleaning & Company Name Simplification:** Cleans company names for informal use.
- **1.10 Lead Enrichment Update & Upload:** Updates enriched data back to Airtable and uploads leads to Instantly for outreach.
- **1.11 Error Handling & Conditional Branching:** Handles empty scrapes, invalid emails, and filtering rejections.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Lead Retrieval

- **Overview:** Accepts incoming webhook calls with lead identifiers, then pulls detailed lead data from Airtable for processing.
- **Nodes Involved:** `Webhook`, `Configure Variables`, `Pull Lead From Airtable`

**Node Details:**

- **Webhook**  
  - Type: Webhook (Trigger)  
  - Role: Entry point for lead processing; expects a record ID to identify the lead.  
  - Configuration: Path set with a unique webhook ID.  
  - Inputs: External webhook calls.  
  - Outputs: Passes record ID to next node.  
  - Failures: Missing/invalid record ID, webhook timeout.

- **Configure Variables**  
  - Type: Set Node  
  - Role: Defines constants such as Airtable Base ID, Table IDs, API limits, and campaign IDs.  
  - Config: User must replace placeholders with real IDs and limits.  
  - Inputs: Triggered by `Webhook`.  
  - Outputs: Provides variables for downstream nodes.  
  - Failures: Misconfiguration leads to invalid API calls.

- **Pull Lead From Airtable**  
  - Type: Airtable (Read)  
  - Role: Fetches the lead record from Airtable using provided record ID.  
  - Config: Reads base and source table from `Configure Variables`.  
  - Inputs: Record ID from webhook.  
  - Outputs: Lead data JSON objects.  
  - Failures: API errors, invalid record ID.

---

#### 1.2 Email Validation

- **Overview:** Validates the lead’s email address via NeverBounce to ensure deliverability before proceeding.
- **Nodes Involved:** `Email Validation - NeverBounce`, `Email Valid?`, `Update Lead - No Valid Email`

**Node Details:**

- **Email Validation - NeverBounce**  
  - Type: HTTP Request  
  - Role: Calls NeverBounce API to check email validity.  
  - Config: Dynamic URL with email parameter; Authorization header with API key required.  
  - Inputs: Email address from Airtable lead.  
  - Outputs: JSON with validation result (`valid`, `invalid`, etc.).  
  - Failures: API errors, rate limits, invalid keys.

- **Email Valid?**  
  - Type: IF Node  
  - Role: Checks if NeverBounce response is `valid`.  
  - Config: String equality check on validation result.  
  - Inputs: NeverBounce response.  
  - Outputs: Branches workflow to continue or mark lead invalid.  
  - Failures: Logic errors if API response format changes.

- **Update Lead - No Valid Email**  
  - Type: Airtable (Update)  
  - Role: Flags lead in Airtable as having an invalid email.  
  - Config: Uses lead record ID to update status or fields.  
  - Inputs: Leads flagged as invalid in email validation.  
  - Outputs: Airtable update confirmation.  
  - Failures: Airtable API errors.

---

#### 1.3 Geographic & Business Model Filtering

- **Overview:** Filters leads to focus only on US-based and B2B companies with employee counts between 5 and 30.
- **Nodes Involved:** `US: Yes or No?`, `Update Lead - Not US`, `Analyze Company LinkedIn_`, `B2B: Yes or No?`, `Update Lead - Not B2B`, `Headcount: >5, <30?`, `Update Lead - Too Big or Small`

**Node Details:**

- **US: Yes or No?**  
  - Type: IF Node  
  - Role: Checks if lead’s location contains “United States”.  
  - Outputs: Continues or updates lead as non-US.  
  - Potential failure: Missing location data.

- **Update Lead - Not US**  
  - Type: Airtable Update  
  - Role: Marks lead as outside US.  
  - Inputs: From US filter negative branch.

- **Analyze Company LinkedIn_**  
  - Type: OpenAI (Langchain)  
  - Role: Analyzes scraped LinkedIn company profile to extract business model info and employee count.  
  - Outputs: JSON with business model (B2B or B2C) and exact employee number.

- **B2B: Yes or No?**  
  - Type: IF Node  
  - Role: Checks if business model is **not** B2C (thus B2B).  
  - Outputs: Continues or updates lead as not B2B.

- **Update Lead - Not B2B**  
  - Type: Airtable Update  
  - Role: Marks lead as not B2B and stops further processing.

- **Headcount: >5, <30?**  
  - Type: IF Node  
  - Role: Filters leads with employee count between 5 and 30 inclusive.  
  - Outputs: Continues or marks lead as too big or small.

- **Update Lead - Too Big or Small**  
  - Type: Airtable Update  
  - Role: Updates lead status for headcount filtering.

---

#### 1.4 Website & Company Data Scraping

- **Overview:** Scrapes company website homepage and LinkedIn company page HTML for data extraction.
- **Nodes Involved:** `Company Homepage Scraper`, `LinkedIn Company Scraper`, `Pull Lead From Airtable` (for URLs)

**Node Details:**

- **Company Homepage Scraper**  
  - Type: HTTP Request  
  - Role: Fetches raw HTML of company homepage, deriving URL from Airtable 'Company Website' or email domain fallback.  
  - Inputs: Website URL or fallback.  
  - Outputs: Raw HTML content.  
  - Failures: Network issues, invalid URLs, timeouts.

- **LinkedIn Company Scraper**  
  - Type: HTTP Request  
  - Role: Fetches LinkedIn company profile HTML based on URL from Airtable.  
  - Inputs: LinkedIn Organization URL.  
  - Outputs: Raw HTML content of LinkedIn page.  
  - Failures: LinkedIn access restrictions, scraping limits, API errors.

---

#### 1.5 HTML Cleaning & Text Extraction

- **Overview:** Cleans HTML content to remove tags, scripts, styles, and decode entities for natural language processing.
- **Nodes Involved:** `Remove HTML Homepage`, `Remove HTML Webpages`, `Remove HTML LinkedIn Page`

**Node Details:**

- **Remove HTML Homepage**  
  - Type: Code Node  
  - Role: Removes HTML tags, scripts, styles from homepage HTML to extract clean text.  
  - Inputs: Raw HTML from homepage scraper.  
  - Outputs: Plain text for AI analysis.  
  - Failures: Empty or malformed HTML.

- **Remove HTML Webpages**  
  - Type: Code Node  
  - Role: Similar cleaning for other webpages or URL content.  
  - Inputs: Aggregated scraped pages.  
  - Outputs: Array of clean texts.  
  - Failures: Similar to above.

- **Remove HTML LinkedIn Page**  
  - Type: Code Node  
  - Role: Cleans LinkedIn company profile HTML for further AI analysis.  
  - Inputs: LinkedIn HTML.  
  - Outputs: Plain text.  
  - Failures: LinkedIn page structure changes.

---

#### 1.6 AI Analysis of Company Data

- **Overview:** Applies multiple GPT-4 prompts to analyze company mission, offerings, processes, and proof of success from cleaned texts.
- **Nodes Involved:** `Analyze Company/Mission`, `Analyze Offerings & Positioning`, `Analyze Process & Differentiation`, `Analyze Proof Of Success`, `Remove Duplicates Texts C/M`, `Remove Duplicates Texts O/P`, `Remove Duplicates Texts P/D`, `Remove Duplicates Texts PoS`

**Node Details:**

- **Remove Duplicates Texts \***  
  - Type: Code Nodes  
  - Role: Merge and deduplicate homepage and webpage texts before AI analysis.  
  - Inputs: Cleaned texts from previous nodes.  
  - Outputs: Final text blocks for each category.  
  - Failures: Text merging logic errors.

- **Analyze Company/Mission**  
  - Type: OpenAI (Langchain)  
  - Role: Extracts verifiable company history, mission, vision, industry served, etc.  
  - Inputs: Merged company/mission text.  
  - Outputs: Structured business overview paragraph.  
  - Failures: AI model errors, prompt failures.

- **Analyze Offerings & Positioning**  
  - Role: Extracts product/service details, pricing, target customers, and use cases.  
  - Similar to above.

- **Analyze Process & Differentiation**  
  - Role: Describes company methodology, proprietary tech, onboarding, value proposition.

- **Analyze Proof Of Success**  
  - Role: Extracts case studies, testimonials, client names, awards, and success metrics.

---

#### 1.7 LinkedIn Profile Scraping & Analysis

- **Overview:** Scrapes LinkedIn personal profiles and posts, analyzes them for decision-maker insights and crafts personalized post-based openers.
- **Nodes Involved:** `LinkedIn Profile Scraper`, `JSON Into Text`, `Analyze Personal LinkedIn`, `Scrape Personal LinkedIn Posts`, `Merge Posts To 1 Item`, `Craft Opening Line - Posts`, `Suitable Posts?`

**Node Details:**

- **LinkedIn Profile Scraper**  
  - Type: HTTP Request (Apify)  
  - Role: Scrapes detailed LinkedIn personal profile JSON.  
  - Inputs: LinkedIn URL from Airtable lead.  
  - Outputs: JSON profile data.  
  - Failures: API limits, scraping failures.

- **JSON Into Text**  
  - Type: Code Node  
  - Role: Converts JSON profile data into text string for AI input.  
  - Outputs: Plain text.

- **Analyze Personal LinkedIn**  
  - Type: OpenAI (Langchain)  
  - Role: Extracts name, expertise, career history, education, and unique value proposition.  
  - Outputs: Structured overview.

- **Scrape Personal LinkedIn Posts**  
  - Type: HTTP Request (Apify)  
  - Role: Retrieves recent LinkedIn posts by the decision-maker.  
  - Inputs: LinkedIn URL.  
  - Outputs: Array of posts.

- **Merge Posts To 1 Item**  
  - Type: Code Node  
  - Role: Merges multiple posts into a single text block.  
  - Outputs: Merged posts text for AI.

- **Craft Opening Line - Posts**  
  - Type: OpenAI (Langchain)  
  - Role: Selects most relevant post, summarizes it, and creates a personalized LinkedIn post opener line.  
  - Outputs: JSON with opener text or no suitable post found.

- **Suitable Posts?**  
  - Type: IF Node  
  - Role: Checks if a suitable post opener was generated.  
  - Branches to update lead or fallback flows.

---

#### 1.8 Personalized Email Content Generation

- **Overview:** Extracts tailored variables using GPT-4 from combined company and personal LinkedIn data to fill cold email templates.
- **Nodes Involved:** `Determine Variables`, `Determine Variables.` (two similar nodes for different data), `Clean Company Name`, `Clean Company Name.`

**Node Details:**

- **Determine Variables**  
  - Type: OpenAI (Langchain)  
  - Role: Extracts company name, first name, specialty, and industry for cold email.  
  - Input: Combined analysis outputs from company and personal LinkedIn nodes.  
  - Output: JSON object with variables.

- **Clean Company Name**  
  - Type: OpenAI (Langchain)  
  - Role: Simplifies company names by removing legal suffixes and generic terms for informal use in emails.  
  - Input: Raw company name from Airtable.  
  - Output: Cleaned company name.

---

#### 1.9 Lead Enrichment Update & Upload

- **Overview:** Updates enriched lead data back into Airtable and uploads leads with personalization data to Instantly for outreach campaigns.
- **Nodes Involved:** `Update Lead W/ Enrichment`, `Update Lead W/ Enrichment.`, `Update Lead W/ Enrichment - Line`, `Update Lead W/ Enrichment. - Line`, `Upload Lead To Instantly`, `Upload Lead To Instantly.`, `Upload Lead To Instantly - Line`, `Upload Lead To Instantly. - Line`

**Node Details:**

- **Update Lead W/ Enrichment** (and variants)  
  - Type: Airtable Update  
  - Role: Stores AI-generated insights, cleaned names, and post openers in the target Airtable table.  
  - Inputs: Enriched data from AI nodes.  
  - Failures: API errors, schema mismatch.

- **Upload Lead To Instantly** (and variants)  
  - Type: HTTP Request  
  - Role: Posts lead data along with custom variables (industry, pain points, company overview) to Instantly API for campaign use.  
  - Requires valid Instantly API key.  
  - Failures: API limits, authentication errors.

---

#### 1.10 Error Handling & Conditional Branching

- **Overview:** Handles scenarios such as empty scrapes, missing posts, invalid emails, and rejects leads not meeting criteria.
- **Nodes Involved:** `Check If Empty`, `Unable To Scrape Website?`, `Posts Available?`, `Posts Available?.`, `Suitable Posts?`, `Suitable Posts?.`

**Node Details:**

- **Check If Empty**  
  - Type: Code Node  
  - Role: Detects empty scraped content and marks for alternate processing.  
  - Outputs: Message flagging empty items.

- **Unable To Scrape Website?**  
  - Type: IF Node  
  - Role: Branches workflow when scraping fails to proceed with alternate logic.

- **Posts Available?** and **Suitable Posts?**  
  - IF Nodes that check presence and suitability of LinkedIn posts for personalized messaging.

---

### 3. Summary Table

| Node Name                     | Node Type                    | Functional Role                                    | Input Node(s)                         | Output Node(s)                         | Sticky Note                                                                                                               |
|-------------------------------|------------------------------|--------------------------------------------------|-------------------------------------|---------------------------------------|---------------------------------------------------------------------------------------------------------------------------|
| TEMPLATE OVERVIEW              | Sticky Note                  | Overview and instructions                         | —                                   | Configure Variables                   | Describes workflow purpose, setup instructions, and requirements.                                                        |
| Configure Variables           | Set                          | Defines constants and API keys                    | Webhook                            | Pull Lead From Airtable                |                                                                                                                           |
| Webhook                      | Webhook Trigger              | Entry point to receive lead record ID             | —                                   | Configure Variables                   | Contains example code snippet for usage.                                                                                   |
| Pull Lead From Airtable       | Airtable Read                | Fetches lead data from Airtable                    | Configure Variables                | Email Validation - NeverBounce         |                                                                                                                           |
| Email Validation - NeverBounce| HTTP Request                 | Checks email validity                              | Pull Lead From Airtable            | Email Valid?                         |                                                                                                                           |
| Email Valid?                 | IF Node                      | Branches on email validity                         | Email Validation - NeverBounce     | US: Yes or No? / Update Lead - No Valid Email |                                                                                                                           |
| Update Lead - No Valid Email | Airtable Update              | Flags lead with invalid email                      | Email Valid? (false branch)        | —                                     |                                                                                                                           |
| US: Yes or No?               | IF Node                      | Filters leads by US location                       | Email Valid? (true branch)         | Wait / Update Lead - Not US            |                                                                                                                           |
| Update Lead - Not US         | Airtable Update              | Flags non-US leads                                 | US: Yes or No? (false branch)      | —                                     |                                                                                                                           |
| Wait                         | Wait Node                   | Adds random delay for rate limit management        | US: Yes or No? (true branch)       | LinkedIn Company Scraper               |                                                                                                                           |
| LinkedIn Company Scraper      | HTTP Request                 | Scrapes LinkedIn company page HTML                 | Wait                             | Remove HTML LinkedIn Page              |                                                                                                                           |
| Remove HTML LinkedIn Page     | Code                        | Cleans LinkedIn company HTML to plain text         | LinkedIn Company Scraper           | Analyze Company LinkedIn_              |                                                                                                                           |
| Analyze Company LinkedIn_     | OpenAI Langchain            | Extracts business model and employee count         | Remove HTML LinkedIn Page          | B2B: Yes or No?                      |                                                                                                                           |
| B2B: Yes or No?              | IF Node                      | Filters non-B2B leads                              | Analyze Company LinkedIn_          | Headcount: >5, <30? / Update Lead - Not B2B |                                                                                                                           |
| Update Lead - Not B2B        | Airtable Update              | Flags leads not B2B                                | B2B: Yes or No? (false branch)     | —                                     |                                                                                                                           |
| Headcount: >5, <30?          | IF Node                      | Filters leads by employee count                    | B2B: Yes or No? (true branch)      | Analyze Company LinkedIn / Update Lead - Too Big or Small |                                                                                                                           |
| Update Lead - Too Big or Small | Airtable Update            | Flags leads outside desired headcount range       | Headcount: >5, <30? (false branch) | —                                     |                                                                                                                           |
| Company Homepage Scraper      | HTTP Request                 | Scrapes company homepage HTML                       | Merge Posts To 1 Item              | Remove HTML Homepage                  |                                                                                                                           |
| Remove HTML Homepage          | Code                        | Cleans homepage HTML to plain text                 | Company Homepage Scraper           | Check If Empty                       |                                                                                                                           |
| Check If Empty               | Code                        | Checks if homepage text is empty                    | Remove HTML Homepage               | Unable To Scrape Website? / Clean Company Name |                                                                                                                           |
| Unable To Scrape Website?    | IF Node                      | Branches on empty homepage text                     | Check If Empty                    | Clean Company Name / Filter HTML For URLs |                                                                                                                           |
| Filter HTML For URLs          | Code                        | Extracts URLs from homepage HTML                    | Unable To Scrape Website?          | Turn X Items Into 1                   |                                                                                                                           |
| Turn X Items Into 1           | Code                        | Merges URL array into single array                  | Filter HTML For URLs               | Determine Valuable URLs               |                                                                                                                           |
| Determine Valuable URLs       | OpenAI Langchain            | Classifies URLs into key company info categories   | Turn X Items Into 1                | Split Out                           |                                                                                                                           |
| Split Out                   | Split Out                   | Splits classified URLs for further processing       | Determine Valuable URLs            | HTTP Request                       |                                                                                                                           |
| HTTP Request                | HTTP Request                 | Fetches content from selected URLs                  | Split Out                       | Aggregate                          |                                                                                                                           |
| Aggregate                   | Aggregate                   | Aggregates webpage content into arrays              | HTTP Request                      | Remove HTML Webpages                |                                                                                                                           |
| Remove HTML Webpages          | Code                        | Cleans webpage HTML content                          | Aggregate                       | Remove Duplicates Texts C/M          |                                                                                                                           |
| Remove Duplicates Texts C/M  | Code                        | Merges homepage and 'Company/Mission' text          | Remove HTML Webpages             | Analyze Company/Mission             |                                                                                                                           |
| Analyze Company/Mission      | OpenAI Langchain            | Analyzes company history, mission, vision           | Remove Duplicates Texts C/M        | Remove Duplicates Texts O/P          |                                                                                                                           |
| Remove Duplicates Texts O/P  | Code                        | Merges offering and positioning text                 | Analyze Company/Mission           | Analyze Offerings & Positioning     |                                                                                                                           |
| Analyze Offerings & Positioning | OpenAI Langchain         | Extracts products, pricing, target customers         | Remove Duplicates Texts O/P        | Remove Duplicates Texts P/D          |                                                                                                                           |
| Remove Duplicates Texts P/D  | Code                        | Merges process and differentiation text              | Analyze Offerings & Positioning   | Analyze Process & Differentiation   |                                                                                                                           |
| Analyze Process & Differentiation | OpenAI Langchain        | Extracts methodology, onboarding, value proposition | Remove Duplicates Texts P/D        | Remove Duplicates Texts PoS          |                                                                                                                           |
| Remove Duplicates Texts PoS  | Code                        | Merges proof of success text                           | Analyze Process & Differentiation | Analyze Proof Of Success            |                                                                                                                           |
| Analyze Proof Of Success     | OpenAI Langchain            | Extracts case studies, testimonials, metrics          | Remove Duplicates Texts PoS        | Clean Company Name.                 |                                                                                                                           |
| Clean Company Name           | OpenAI Langchain            | Cleans company name for informal usage                | Analyze Proof Of Success          | Determine Variables                 |                                                                                                                           |
| Determine Variables          | OpenAI Langchain            | Extracts cold email variables from company and personal data | Clean Company Name              | Posts Available?                   |                                                                                                                           |
| Posts Available?            | IF Node                      | Checks if LinkedIn posts exist                         | Determine Variables              | Craft Opening Line - Posts / Update Lead W/ Enrichment |                                                                                                                           |
| Scrape Personal LinkedIn Posts | HTTP Request             | Scrapes personal LinkedIn posts                        | Analyze Personal LinkedIn         | Merge Posts To 1 Item               |                                                                                                                           |
| Merge Posts To 1 Item        | Code                        | Merges posts into single text block                    | Scrape Personal LinkedIn Posts   | Craft Opening Line - Posts         |                                                                                                                           |
| Craft Opening Line - Posts   | OpenAI Langchain            | Selects and summarizes best LinkedIn post for email opener | Merge Posts To 1 Item           | Suitable Posts?                   |                                                                                                                           |
| Suitable Posts?             | IF Node                      | Checks suitability of LinkedIn post opener             | Craft Opening Line - Posts       | Update Lead W/ Enrichment - Line / Update Lead W/ Enrichment |                                                                                                                           |
| Update Lead W/ Enrichment    | Airtable Update              | Updates Airtable with enriched data                     | Suitable Posts?                  | Upload Lead To Instantly           |                                                                                                                           |
| Upload Lead To Instantly     | HTTP Request                 | Uploads enriched lead and personalization to Instantly  | Update Lead W/ Enrichment        | —                                 |                                                                                                                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: Webhook Trigger  
   - Set unique path (e.g., `698bc83b-22b9-412f-894b-f74554964bcb`).  
   - This node receives incoming requests with a `recordId` parameter.

2. **Create Set Node "Configure Variables"**  
   - Define string variables:  
     - `airtableBaseId` = YOUR_AIRTABLE_BASE_ID  
     - `airtableSourceTableId` = YOUR_AIRTABLE_SOURCE_TABLE_ID  
     - `airtableTargetTableId` = YOUR_AIRTABLE_TARGET_TABLE_ID  
     - `apifyPostsLimit` = 10 (or desired limit)  
     - `instantlyCampaignId` = YOUR_INSTANTLY_CAMPAIGN_ID  
   - Connect Webhook → Configure Variables.

3. **Add Airtable Read Node "Pull Lead From Airtable"**  
   - Credentials: Airtable Personal Access Token.  
   - Base ID: `={{ $node["Configure Variables"].json.airtableBaseId }}`  
   - Table ID: `={{ $node["Configure Variables"].json.airtableSourceTableId }}`  
   - Query with recordId from webhook.  
   - Connect Configure Variables → Pull Lead From Airtable.

4. **Add HTTP Request Node "Email Validation - NeverBounce"**  
   - Method: GET  
   - URL: `https://api.neverbounce.com/v4/single/check?email={{ $json["Email Address"] }}`  
   - Header: Authorization Bearer with NeverBounce API key.  
   - Connect Pull Lead From Airtable → Email Validation - NeverBounce.

5. **Add IF Node "Email Valid?"**  
   - Condition: `result` equals "valid".  
   - True → continue to geographic filtering; False → go to "Update Lead - No Valid Email".

6. **Add Airtable Update Node "Update Lead - No Valid Email"**  
   - Update lead record with invalid email flag.  
   - Connect Email Valid? (false) → Update Lead - No Valid Email.

7. **Add IF Node "US: Yes or No?"**  
   - Check if `Location` field contains "United States".  
   - True → Wait node; False → Update Lead - Not US.

8. **Add Airtable Update Node "Update Lead - Not US"**  
   - Marks lead as outside US.  
   - Connect US: Yes or No? (false) → Update Lead - Not US.

9. **Add Wait Node "Wait"**  
   - Random delay between 8-20 seconds.  
   - Connect US: Yes or No? (true) → Wait.

10. **Add HTTP Request Node "LinkedIn Company Scraper"**  
    - URL from Airtable field `LinkedIn Organization URL`.  
    - Connect Wait → LinkedIn Company Scraper.

11. **Add Code Node "Remove HTML LinkedIn Page"**  
    - Script removes scripts, styles, HTML tags, decodes entities.  
    - Connect LinkedIn Company Scraper → Remove HTML LinkedIn Page.

12. **Add OpenAI Node "Analyze Company LinkedIn_"**  
    - Prompt to extract business model and employee number.  
    - Connect Remove HTML LinkedIn Page → Analyze Company LinkedIn_.

13. **Add IF Node "B2B: Yes or No?"**  
    - Condition: Business Model not equals "B2C".  
    - True → Headcount filter; False → Update Lead - Not B2B.

14. **Add Airtable Update Node "Update Lead - Not B2B"**  
    - Marks lead as not B2B.  
    - Connect B2B: Yes or No? (false) → Update Lead - Not B2B.

15. **Add IF Node "Headcount: >5, <30?"**  
    - Checks employee count between 5 and 30 inclusive.  
    - True → continue; False → Update Lead - Too Big or Small.

16. **Add Airtable Update Node "Update Lead - Too Big or Small"**  
    - Flags leads with unsuitable headcount.  
    - Connect Headcount check false → Update Lead - Too Big or Small.

17. **Add HTTP Request Node "Company Homepage Scraper"**  
    - URL from `Company Website` or derived from email domain.  
    - Connect Headcount check true → Company Homepage Scraper.

18. **Add Code Node "Remove HTML Homepage"**  
    - Clean homepage HTML to plain text.  
    - Connect Company Homepage Scraper → Remove HTML Homepage.

19. **Add Code Node "Check If Empty"**  
    - Checks if plain text is empty.  
    - True → Unable To Scrape Website? (yes branch); False → Clean Company Name.

20. **Add IF Node "Unable To Scrape Website?"**  
    - Branch based on empty homepage text.  
    - True → Clean Company Name node; False → Filter HTML For URLs.

21. **Add Code Node "Filter HTML For URLs"**  
    - Extracts URLs from homepage HTML.  
    - Connect Unable To Scrape Website? (false) → Filter HTML For URLs.

22. **Add Code Node "Turn X Items Into 1"**  
    - Merges URL objects into a list of URLs.  
    - Connect Filter HTML For URLs → Turn X Items Into 1.

23. **Add OpenAI Node "Determine Valuable URLs"**  
    - Classifies URLs into 4 categories per prompt.  
    - Connect Turn X Items Into 1 → Determine Valuable URLs.

24. **Add Split Out Node "Split Out"**  
    - Splits URLs by category for fetching.  
    - Connect Determine Valuable URLs → Split Out.

25. **Add HTTP Request Node "HTTP Request"**  
    - Fetches content from selected URLs (dynamic URL construction).  
    - Connect Split Out → HTTP Request.

26. **Add Aggregate Node "Aggregate"**  
    - Aggregates fetched webpage content arrays.  
    - Connect HTTP Request → Aggregate.

27. **Add Code Node "Remove HTML Webpages"**  
    - Cleans fetched webpage HTML content.  
    - Connect Aggregate → Remove HTML Webpages.

28. **Add Code Nodes to Remove Duplicate Texts per Category**  
    - For Company/Mission, Offerings/Positioning, Process/Differentiation, Proof of Success.  
    - Chain: Remove HTML Webpages → Remove Duplicates Texts C/M → Analyze Company/Mission → Remove Duplicates Texts O/P → Analyze Offerings & Positioning → Remove Duplicates Texts P/D → Analyze Process & Differentiation → Remove Duplicates Texts PoS → Analyze Proof Of Success  
    - Each Analyze node is an OpenAI Langchain call with specific prompts.

29. **Add OpenAI Node "Clean Company Name"**  
    - Cleans company name by removing suffixes and generic terms.  
    - Connect Analyze Proof Of Success → Clean Company Name.

30. **Add OpenAI Node "Determine Variables"**  
    - Extracts personalized cold email variables from combined company and personal LinkedIn data.  
    - Connect Clean Company Name → Determine Variables.

31. **Add HTTP Request Node "LinkedIn Profile Scraper"**  
    - Scrapes LinkedIn personal profile JSON from URL.  
    - Connect Analyze Company LinkedIn_ → LinkedIn Profile Scraper.

32. **Add Code Node "JSON Into Text"**  
    - Converts JSON profile data to plain text.  
    - Connect LinkedIn Profile Scraper → JSON Into Text.

33. **Add OpenAI Node "Analyze Personal LinkedIn"**  
    - Extracts decision-maker profile summary.  
    - Connect JSON Into Text → Analyze Personal LinkedIn.

34. **Add HTTP Request Node "Scrape Personal LinkedIn Posts"**  
    - Retrieves recent LinkedIn posts (limit from variables).  
    - Connect Analyze Personal LinkedIn → Scrape Personal LinkedIn Posts.

35. **Add Code Node "Merge Posts To 1 Item"**  
    - Merges multiple posts into one text block.  
    - Connect Scrape Personal LinkedIn Posts → Merge Posts To 1 Item.

36. **Add OpenAI Node "Craft Opening Line - Posts"**  
    - Filters and selects best post to create personalized cold email opener line.  
    - Connect Merge Posts To 1 Item → Craft Opening Line - Posts.

37. **Add IF Node "Suitable Posts?"**  
    - Checks if a suitable post opener was created.  
    - True → Update Lead W/ Enrichment - Line / Upload Lead To Instantly - Line  
    - False → Update Lead W/ Enrichment / Upload Lead To Instantly (fallback).

38. **Add Airtable Update Nodes for Enrichment**  
    - Update lead with enriched data and personalized opener.  
    - Connect Suitable Posts? branches accordingly.

39. **Add HTTP Request Nodes "Upload Lead To Instantly"**  
    - POST enriched lead to Instantly API with campaign ID and personalization variables.  
    - Connect Airtable updates → Upload Lead To Instantly nodes.

---

### 5. General Notes & Resources

| Note Content                                                                                                                          | Context or Link                                                                                              |
|----------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| Workflow automates advanced lead enrichment by combining Airtable, LinkedIn company pages, decision-maker profiles, and AI insights.  | Workflow Overview Sticky Note.                                                                               |
| OpenAI GPT-4 model "gpt-4.1-mini" is used for all AI analysis nodes for structured business insights and cold email personalization. | Multiple OpenAI node configurations.                                                                        |
| Apify scrapers for LinkedIn company profile and personal LinkedIn posts are used; ensure valid API tokens and scrape limits.           | Sticky Notes with Apify scraper links: https://console.apify.com/actors/PEgClm7RgRD7YO94b/input and https://console.apify.com/actors/LQQIXN9Othf8f7R5n/input |
| Airtable credentials require Personal Access Token with appropriate base and table access.                                              | Airtable nodes credentials.                                                                                  |
| NeverBounce API key required for email validation node.                                                                                 | Email Validation - NeverBounce node.                                                                          |
| Instantly API key and campaign ID needed to upload enriched leads for outreach campaigns.                                                | Upload Lead To Instantly nodes.                                                                               |
| Several sticky notes explain expected data extraction from LinkedIn sections (header, about, sidebar, experience, education).          | Sticky Notes 2, 3, 4, 5, 6, 9, 10, 11, 12 provide detailed guidance on data extraction.                      |
| Careful handling of empty scrapes, unsuitable posts, and invalid emails ensures data quality and workflow robustness.                  | Conditional nodes and error handling logic.                                                                  |
| The workflow includes multiple points where user configuration is necessary (API keys, Airtable IDs, campaign IDs).                    | See TEMPLATE OVERVIEW sticky note for setup instructions.                                                    |

---

**Disclaimer:** The text documented here is based exclusively on an automated n8n workflow. It complies with all content policies and contains no illegal or protected data. All processed data is publicly available and legal.