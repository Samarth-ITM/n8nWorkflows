Qualify & Enrich Leads with GPT-4 and LinkedIn Data for Intelligent Routing

https://n8nworkflows.xyz/workflows/qualify---enrich-leads-with-gpt-4-and-linkedin-data-for-intelligent-routing-11493


# Qualify & Enrich Leads with GPT-4 and LinkedIn Data for Intelligent Routing

### 1. Workflow Overview

This workflow automates the lead qualification, enrichment, and routing process for an AI Automation Agency using GPT-4 and LinkedIn data. It is designed for businesses receiving inbound leads via a web form and aims to intelligently score and enrich those leads to optimize sales outreach and CRM integration.

The workflow logic is organized into the following functional blocks:

- **1.1 Lead Capture & Validation**  
  Captures lead data from a web form, normalizes and validates key fields, including business vs personal email detection.

- **1.2 Company Enrichment via LinkedIn (Apify Scrapers)**  
  For business emails, performs a Google search to find the company LinkedIn URL, scrapes LinkedIn company profile data, and merges it with the lead profile.

- **1.3 AI-Based Lead Qualification (GPT-4)**  
  Uses OpenAI GPT-4 to assess the lead’s quality, score (0-100), tier (Hot/Warm/Cold/Disqualified), buyer persona, and generate insights for sales outreach.

- **1.4 Lead Routing & Actions**  
  Routes leads based on qualification tier to Slack alerts, CRM (HubSpot), Google Sheets logging, and personalized email outreach for hot leads.

---

### 2. Block-by-Block Analysis

#### 1.1 Lead Capture & Validation

**Overview:**  
Captures lead data via a web form, normalizes fields, validates email and required inputs, and determines if the email is business or personal. Personal emails skip enrichment.

**Nodes Involved:**
- Lead Capture Form
- Send Form Response
- Normalize Lead Data1
- Valid Lead?1
- Business Email?
- Handle Invalid Lead (error stop)

**Node Details:**

- **Lead Capture Form**  
  *Type:* Form Trigger  
  *Role:* Entry point — shows a form with fields (Email, First Name, Last Name, Phone, Company, Job Title, Message, Budget, Timeline, Lead Source).  
  *Config:* Validations on required fields Email, First Name, and message. Response mode set to respond immediately.  
  *Input/Output:* Trigger webhook → leads to Send Form Response and Normalize Lead Data1.  
  *Failures:* Form submission errors, webhook failures.

- **Send Form Response**  
  *Type:* Respond to Webhook  
  *Role:* Sends immediate thank you response to form submitter.  
  *Config:* Sends plain text confirmation message.  
  *Input/Output:* Input from Lead Capture Form → no outputs (end of response).  
  *Failures:* None typical.

- **Normalize Lead Data1**  
  *Type:* Code  
  *Role:* Extracts and normalizes lead data from form input; creates a consistent lead object with trimmed and lowercased fields, extracts email domain, detects business email, validates required fields.  
  *Key Expressions:* Checks personal email domains to flag business email; sets isValid boolean and validationErrors array.  
  *Input/Output:* Input from Lead Capture Form → output to Valid Lead?1.  
  *Failures:* Logic errors in parsing, malformed inputs.

- **Valid Lead?1**  
  *Type:* If  
  *Role:* Branches workflow based on whether the lead passed validation (email format, first name presence).  
  *Input/Output:* Input from Normalize Lead Data1 → main (true) to Business Email?; alternate to Handle Invalid Lead.  
  *Failures:* Expression evaluation errors possible.

- **Business Email?**  
  *Type:* If  
  *Role:* Checks if email is business type for enrichment eligibility.  
  *Input/Output:* Input from Valid Lead?1 → main (true) to Google Search (Find LinkedIn URL); false to Skip Enrichment (Fallback).  
  *Failures:* Expression failures.

- **Handle Invalid Lead**  
  *Type:* Stop and Error  
  *Role:* Stops workflow with error message detailing validation issues.  
  *Input/Output:* Input from Valid Lead?1 alternate branch; ends workflow with error.  
  *Failures:* N/A (intended for error handling).

---

#### 1.2 Company Enrichment via LinkedIn (Apify)

**Overview:**  
Performs enrichment of lead’s company data by searching for LinkedIn company URL using Google search scraping, then scraping LinkedIn profile data, and merging results.

**Nodes Involved:**
- Google Search (Find LinkedIn URL)
- Extract LinkedIn URL (Code)
- LinkedIn URL Found? (If)
- LinkedIn Company Scraper
- Merge LinkedIn Data (Code)
- Skip Enrichment (Fallback)

**Node Details:**

- **Google Search (Find LinkedIn URL)**  
  *Type:* HTTP Request  
  *Role:* Calls Apify Google Search Scraper with a query to find LinkedIn company page URL for the lead’s company or email domain.  
  *Config:* POST request with dynamic JSON body using company/email domain; includes authorization header requiring Apify token.  
  *Input/Output:* Input from Business Email?; output to Extract LinkedIn URL.  
  *Failures:* API authentication errors, rate limits, network timeouts.

- **Extract LinkedIn URL**  
  *Type:* Code  
  *Role:* Parses Google search results to find and validate LinkedIn company URL based on company name and email domain heuristics; sets flags if found.  
  *Key Expressions:* Regex matching LinkedIn URL patterns; attempts matching by slug or title; fallback to first valid URL.  
  *Input/Output:* Input from Google Search; output to LinkedIn URL Found?.  
  *Failures:* Parsing errors, unexpected response formats.

- **LinkedIn URL Found?**  
  *Type:* If  
  *Role:* Checks if LinkedIn URL was successfully found from search results.  
  *Input/Output:* Input from Extract LinkedIn URL; true to LinkedIn Company Scraper, false to Skip Enrichment (Fallback).  
  *Failures:* Expression evaluation failures.

- **LinkedIn Company Scraper**  
  *Type:* HTTP Request  
  *Role:* Calls Apify LinkedIn Company Scraper to extract detailed company profile data from the LinkedIn URL.  
  *Config:* POST request with JSON body containing LinkedIn URL; requires Apify authorization token.  
  *Input/Output:* Input from LinkedIn URL Found?; output to Merge LinkedIn Data.  
  *Failures:* API errors, rate limits, invalid URL.

- **Merge LinkedIn Data**  
  *Type:* Code  
  *Role:* Merges LinkedIn company data with normalized lead profile; extracts multiple fields like industry, size, location, specialties, founded year, logo, similar companies.  
  *Input/Output:* Input from LinkedIn Company Scraper; output to AI Lead Qualification1.  
  *Failures:* Parsing errors, missing fields.

- **Skip Enrichment (Fallback)**  
  *Type:* Code  
  *Role:* Generates a lead profile without enrichment for cases where LinkedIn URL not found or email is personal; marks enrichment as skipped with reasons.  
  *Input/Output:* Input from Business Email? (false) or LinkedIn URL Found? (false); output to AI Lead Qualification1.  
  *Failures:* None expected.

---

#### 1.3 AI-Based Lead Qualification (GPT-4)

**Overview:**  
Uses OpenAI GPT-4 via Langchain node to analyze lead and company data to produce a qualification score, tier, buyer persona, urgency, budget fit, insights, suggested next steps, talking points, risk factors, and outreach recommendations.

**Nodes Involved:**
- AI Lead Qualification1
- Build Qualified Lead Profile1

**Node Details:**

- **AI Lead Qualification1**  
  *Type:* OpenAI (Langchain)  
  *Role:* Sends detailed system prompt and formatted lead data to GPT-4 model to receive JSON qualification output.  
  *Config:* Model GPT-4.1, temperature 0.3 for consistent output; system message defines exact JSON schema and scoring guidelines.  
  *Input/Output:* Input from Merge LinkedIn Data or Skip Enrichment; output to Build Qualified Lead Profile1.  
  *Failures:* API errors, invalid JSON parsing, rate limits.

- **Build Qualified Lead Profile1**  
  *Type:* Code  
  *Role:* Parses AI response JSON; combines AI qualification data with lead & enrichment info to build a comprehensive qualified lead object including metadata and outreach recommendations.  
  *Input/Output:* Input from AI Lead Qualification1; output to Hot Lead?1 and Log to Google Sheets1.  
  *Failures:* JSON parsing errors, fallback default qualification on failure.

---

#### 1.4 Lead Routing & Actions

**Overview:**  
Routes leads based on qualification tier with specific actions for Hot, Warm, and Cold leads: Slack alerts, personalized emails, CRM contact creation, and logging to Google Sheets.

**Nodes Involved:**
- Hot Lead?1
- Slack Hot Lead Alert1
- AI Generate Personalized Email1
- Create HubSpot Contact1
- Warm Lead?1
- Slack Warm Lead Notification1
- Slack Cold Lead Log
- Log to Google Sheets1
- Send Personalized Email

**Node Details:**

- **Hot Lead?1**  
  *Type:* If  
  *Role:* Checks if lead tier is "hot".  
  *Input/Output:* Input from Build Qualified Lead Profile1; true to Slack Hot Lead Alert1 then AI Generate Personalized Email1; false to Warm Lead?1.  
  *Failures:* Expression evaluation errors.

- **Slack Hot Lead Alert1**  
  *Type:* Slack  
  *Role:* Sends detailed Slack alert message about the hot lead to specified channel with lead info and AI insights.  
  *Config:* OAuth2 authentication; channel ID set; message template includes rich info.  
  *Input/Output:* Input from Hot Lead?1; output to Create HubSpot Contact1.  
  *Failures:* Slack API errors, OAuth token expiry.

- **AI Generate Personalized Email1**  
  *Type:* OpenAI (Langchain)  
  *Role:* Generates personalized outreach email content for hot leads based on lead and AI insights.  
  *Config:* GPT-4.1, temperature 0.7 for creative email; system prompt instructs professional, concise email.  
  *Input/Output:* Input from Slack Hot Lead Alert1; output to Send Personalized Email.  
  *Failures:* API errors, invalid JSON.

- **Create HubSpot Contact1**  
  *Type:* HubSpot  
  *Role:* Creates or updates HubSpot contact with lead details, score, tier, and AI insights.  
  *Config:* Uses HubSpot App Token credentials; maps multiple custom properties.  
  *Input/Output:* Input from Slack Hot Lead Alert1; no output (end node).  
  *Failures:* API authentication, rate limits, missing required fields.

- **Warm Lead?1**  
  *Type:* If  
  *Role:* Checks if lead tier is "warm".  
  *Input/Output:* Input from Hot Lead?1 false branch; true to Slack Warm Lead Notification1; false to Slack Cold Lead Log.  
  *Failures:* Expression evaluation errors.

- **Slack Warm Lead Notification1**  
  *Type:* Slack  
  *Role:* Sends a brief notification to Slack about warm leads added to nurture sequences.  
  *Config:* OAuth2 auth; channel ID set; concise message including lead score and snippet of message.  
  *Input/Output:* Input from Warm Lead?1; no output.  
  *Failures:* Slack API errors.

- **Slack Cold Lead Log**  
  *Type:* Slack  
  *Role:* Logs cold or disqualified leads with reasoning to Slack channel for record keeping.  
  *Config:* OAuth2 auth; channel ID set; message with score, tier, and reason.  
  *Input/Output:* Input from Warm Lead?1 false branch; no output.  
  *Failures:* Slack API errors.

- **Log to Google Sheets1**  
  *Type:* Google Sheets  
  *Role:* Appends all lead data, qualification, enrichment, and AI insights to a Google Sheet for tracking and analysis.  
  *Config:* OAuth2 credentials; schema defines columns covering contact, company, AI insights, outreach info, etc.  
  *Input/Output:* Input from Build Qualified Lead Profile1; no output.  
  *Failures:* API quota, permissions.

- **Send Personalized Email**  
  *Type:* Gmail  
  *Role:* Sends AI-generated personalized outreach email to hot lead.  
  *Config:* OAuth2 Gmail credentials; sends email with subject and body from AI Generate Personalized Email1.  
  *Input/Output:* Input from AI Generate Personalized Email1; no output.  
  *Failures:* SMTP errors, OAuth token expiration.

---

### 3. Summary Table

| Node Name                    | Node Type                  | Functional Role                                 | Input Node(s)             | Output Node(s)                  | Sticky Note                                                                                                  |
|------------------------------|----------------------------|------------------------------------------------|---------------------------|---------------------------------|--------------------------------------------------------------------------------------------------------------|
| Lead Capture Form             | Form Trigger               | Capture lead input from web form                | -                         | Send Form Response, Normalize Lead Data1 | See Sticky Note: Intro, AI-Powered Lead Qualification overview                                               |
| Send Form Response            | Respond to Webhook         | Send confirmation response to lead              | Lead Capture Form          | -                               |                                                                                                              |
| Normalize Lead Data1          | Code                       | Normalize & validate lead data                   | Lead Capture Form          | Valid Lead?1                    | Sticky Note1: Lead Capture & Validation details                                                             |
| Valid Lead?1                 | If                         | Branch on lead validity (valid or not)          | Normalize Lead Data1       | Business Email?, Handle Invalid Lead |                                                                                                              |
| Business Email?               | If                         | Check if email is business type                  | Valid Lead?1               | Google Search or Skip Enrichment | Sticky Note2: Company Enrichment (Apify)                                                                     |
| Google Search (Find LinkedIn URL) | HTTP Request             | Search LinkedIn company page URL via Apify      | Business Email?            | Extract LinkedIn URL            | See Sticky Note2                                                                                            |
| Extract LinkedIn URL          | Code                       | Extract LinkedIn company URL from search results| Google Search              | LinkedIn URL Found?             |                                                                                                              |
| LinkedIn URL Found?           | If                         | Check if LinkedIn URL found                       | Extract LinkedIn URL       | LinkedIn Company Scraper, Skip Enrichment |                                                                                                              |
| LinkedIn Company Scraper      | HTTP Request               | Scrape company profile data from LinkedIn       | LinkedIn URL Found?        | Merge LinkedIn Data             |                                                                                                              |
| Merge LinkedIn Data           | Code                       | Merge scraped LinkedIn data with lead profile   | LinkedIn Company Scraper   | AI Lead Qualification1          | Sticky Note3: AI Qualification (GPT-4)                                                                       |
| Skip Enrichment (Fallback)    | Code                       | Create lead profile without enrichment           | Business Email?, LinkedIn URL Found? | AI Lead Qualification1          |                                                                                                              |
| AI Lead Qualification1       | OpenAI (Langchain)         | Use GPT-4 to qualify lead & generate insights    | Merge LinkedIn Data, Skip Enrichment | Build Qualified Lead Profile1 | See Sticky Note3                                                                                            |
| Build Qualified Lead Profile1 | Code                       | Compile final qualified lead object with AI data| AI Lead Qualification1     | Hot Lead?1, Log to Google Sheets1 | Sticky Note4: Lead Routing & Actions                                                                        |
| Hot Lead?1                   | If                         | Check if lead is hot tier                         | Build Qualified Lead Profile1 | Slack Hot Lead Alert1, Warm Lead?1 | Sticky Note5: Hot Lead Actions                                                                               |
| Slack Hot Lead Alert1         | Slack                      | Send detailed Slack alert for hot leads          | Hot Lead?1                | Create HubSpot Contact1          | See Sticky Note5                                                                                            |
| AI Generate Personalized Email1 | OpenAI (Langchain)       | Generate personalized outreach email for hot leads | Slack Hot Lead Alert1      | Send Personalized Email          |                                                                                                              |
| Create HubSpot Contact1       | HubSpot                    | Create/update contact in HubSpot CRM              | Slack Hot Lead Alert1      | -                               |                                                                                                              |
| Send Personalized Email       | Gmail                      | Send AI-generated personalized email              | AI Generate Personalized Email1 | -                           |                                                                                                              |
| Warm Lead?1                  | If                         | Check if lead is warm tier                        | Hot Lead?1                | Slack Warm Lead Notification1, Slack Cold Lead Log |                                                                                                              |
| Slack Warm Lead Notification1 | Slack                      | Notify Slack channel of new warm lead             | Warm Lead?1               | -                               |                                                                                                              |
| Slack Cold Lead Log           | Slack                      | Log cold/disqualified leads in Slack              | Warm Lead?1               | -                               |                                                                                                              |
| Log to Google Sheets1         | Google Sheets              | Append lead data and insights to Google Sheet     | Build Qualified Lead Profile1 | -                             |                                                                                                              |
| Handle Invalid Lead           | Stop and Error             | Stop workflow on invalid lead with error message | Valid Lead?1 (false branch) | -                             |                                                                                                              |
| Sticky Note                  | Sticky Note                | Documentation and workflow overview                | -                         | -                               | See content in sticky notes sections above                                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger Node (`Lead Capture Form`):**  
   - Set webhook path to `lead-capture-form`.  
   - Configure form fields: Email (required, type email), First Name (required), Last Name, Phone, Company, Job Title, Message (required textarea), Budget Range (dropdown), Timeline (dropdown), Lead Source (dropdown).  
   - Set response mode to respond with a node.

2. **Add Respond to Webhook Node (`Send Form Response`):**  
   - Connect from `Lead Capture Form`.  
   - Configure response body: "Thanks for your submission! We'll be in touch soon."  
   - Set response type to text.

3. **Add Code Node (`Normalize Lead Data1`):**  
   - Connect from `Lead Capture Form`.  
   - Write JavaScript to normalize input fields: lowercase email, trim strings, extract email domain, identify if business email, build fullName, validate email and firstName, store raw form data.  
   - Output normalized lead object with flags: isValid, isBusinessEmail, validationErrors.

4. **Add If Node (`Valid Lead?1`):**  
   - Connect from `Normalize Lead Data1`.  
   - Condition: `$json.isValid` equals true.  
   - True path continues, false path to error handling.

5. **Add Stop and Error Node (`Handle Invalid Lead`):**  
   - Connect from `Valid Lead?1` false path.  
   - Use error message with validation error details.

6. **Add If Node (`Business Email?`):**  
   - Connect from `Valid Lead?1` true path.  
   - Condition: `$json.isBusinessEmail` equals true.  
   - True path to enrichment; false path to skip enrichment.

7. **Add HTTP Request Node (`Google Search (Find LinkedIn URL)`):**  
   - Connect from `Business Email?` true path.  
   - POST request to Apify Google Search Scraper endpoint.  
   - JSON body includes search term with company name or email domain.  
   - Add header with Apify token authorization.  
   - Configure to continue on failure to avoid workflow break.

8. **Add Code Node (`Extract LinkedIn URL`):**  
   - Connect from `Google Search`.  
   - JavaScript to parse search results, identify valid LinkedIn company URL using regex and string matching heuristics.

9. **Add If Node (`LinkedIn URL Found?`):**  
   - Connect from `Extract LinkedIn URL`.  
   - Condition: `$json.linkedInSearchSuccess` equals true.  
   - True path to LinkedIn scraping; false to skip enrichment fallback.

10. **Add HTTP Request Node (`LinkedIn Company Scraper`):**  
    - Connect from `LinkedIn URL Found?` true path.  
    - POST to Apify LinkedIn Company Scraper endpoint with LinkedIn URL in JSON body.  
    - Include Apify authorization token.

11. **Add Code Node (`Merge LinkedIn Data`):**  
    - Connect from `LinkedIn Company Scraper`.  
    - Parse LinkedIn data, extract company fields (industry, size, specialties, founded year, logo, similar companies).  
    - Merge with normalized lead data.

12. **Add Code Node (`Skip Enrichment (Fallback)`):**  
    - Connect from `Business Email?` false path and `LinkedIn URL Found?` false path.  
    - Build lead profile with empty enrichment fields, mark enrichmentSkipped with reason.

13. **Add OpenAI Node (`AI Lead Qualification1`):**  
    - Connect from `Merge LinkedIn Data` and `Skip Enrichment (Fallback)`.  
    - Use GPT-4.1 model, temperature 0.3.  
    - System prompt defines JSON output schema and scoring rules.  
    - Message includes full lead and enrichment data for assessment.  
    - Output JSON parsed automatically.

14. **Add Code Node (`Build Qualified Lead Profile1`):**  
    - Connect from `AI Lead Qualification1`.  
    - Parse AI JSON response safely; fallback on parse errors.  
    - Construct final lead object with contact info, enrichment, AI qualification, insights, outreach recommendations.

15. **Add If Node (`Hot Lead?1`):**  
    - Connect from `Build Qualified Lead Profile1`.  
    - Condition: Qualification tier equals "hot".  
    - True path to Slack alert and AI email generation; false to check warm lead.

16. **Add Slack Node (`Slack Hot Lead Alert1`):**  
    - Connect from `Hot Lead?1` true path.  
    - OAuth2 authenticated Slack message to a channel with detailed lead info and AI insights.

17. **Add OpenAI Node (`AI Generate Personalized Email1`):**  
    - Connect from `Slack Hot Lead Alert1`.  
    - GPT-4.1, temperature 0.7.  
    - System prompt directs email writing style and structure based on lead data.

18. **Add Gmail Node (`Send Personalized Email`):**  
    - Connect from `AI Generate Personalized Email1`.  
    - OAuth2 Gmail credentials.  
    - Send email to lead address with subject and body from AI output.

19. **Add HubSpot Node (`Create HubSpot Contact1`):**  
    - Connect from `Slack Hot Lead Alert1`.  
    - Use HubSpot App Token credentials.  
    - Map lead fields and AI qualification as custom properties.

20. **Add If Node (`Warm Lead?1`):**  
    - Connect from `Hot Lead?1` false path.  
    - Condition: Qualification tier equals "warm".  
    - True path to Slack warm notification; false path to cold lead log.

21. **Add Slack Node (`Slack Warm Lead Notification1`):**  
    - Connect from `Warm Lead?1` true path.  
    - OAuth2 Slack message to channel with brief warm lead details.

22. **Add Slack Node (`Slack Cold Lead Log`):**  
    - Connect from `Warm Lead?1` false path.  
    - OAuth2 Slack message logging cold or disqualified leads with reasons.

23. **Add Google Sheets Node (`Log to Google Sheets1`):**  
    - Connect from `Build Qualified Lead Profile1`.  
    - OAuth2 Google Sheets credentials.  
    - Append lead and AI data to a spreadsheet with defined columns.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                | Context or Link                                                                                           |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| ## 🚀 AI-Powered Lead Qualification Template Overview  This workflow automatically qualifies, enriches, and routes leads using AI and LinkedIn data. Requires OpenAI, Apify, Slack, HubSpot, Gmail, Google Sheets credentials. Cost ~$0.02-0.03 per enriched lead. For help, visit the [n8n Discord](https://discord.gg/n8n) or contact [Agentical AI](https://agenticalai.com).                                            | Sticky Note at workflow start                                                                             |
| Lead Capture & Validation includes email format checks, business vs personal email detection, and normalization. Personal emails skip enrichment but still get AI qualification.                                                                                                                                                                                                                                              | Sticky Note1                                                                                              |
| Company enrichment uses Apify Google Search Scraper and LinkedIn Company Scraper PPR actors to fetch LinkedIn company URLs and profiles based on lead company name or email domain.                                                                                                                                                                                                                                         | Sticky Note2                                                                                              |
| AI Qualification uses GPT-4 to score leads 0-100 and assign tiers with detailed insights, talking points, risk factors, and outreach recommendations. The system prompt is editable for custom scoring criteria.                                                                                                                                                                                                           | Sticky Note3                                                                                              |
| Lead Routing Actions: Hot leads trigger Slack alerts, personalized AI emails, and HubSpot contact creation; Warm leads get Slack notifications; Cold leads are logged to Slack; All leads logged to Google Sheets.                                                                                                                                                                                                           | Sticky Note4                                                                                              |
| Hot Lead Actions include instant Slack notification with lead details, AI-generated personalized outreach email, and HubSpot CRM integration.                                                                                                                                                                                                                                                                            | Sticky Note5                                                                                              |
| Apify API credentials setup: Use "HTTP Query Auth" with parameter name `token` and your Apify API key as value.                                                                                                                                                                                                                                                                                                           | In Sticky Note content                                                                                     |
| Ensure all credentials (OpenAI API, Slack OAuth2, HubSpot App Token, Gmail OAuth2, Google Sheets OAuth2, Apify token) are configured before activating workflow.                                                                                                                                                                                                                                                             | General prerequisite                                                                                       |
| The workflow handles errors gracefully: invalid leads are stopped with error messages; API failures in enrichment proceed with fallback; Slack notifications and logging maintain visibility of all lead states.                                                                                                                                                                                                           | Observed error handling strategy                                                                          |

---

**Disclaimer:** The text provided is extracted exclusively from an automated n8n workflow. This workflow respects all applicable content policies and contains no illegal, offensive, or protected elements. All processed data are legal and publicly available.