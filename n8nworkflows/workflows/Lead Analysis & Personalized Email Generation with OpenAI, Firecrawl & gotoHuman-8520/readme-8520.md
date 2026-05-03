Lead Analysis & Personalized Email Generation with OpenAI, Firecrawl & gotoHuman

https://n8nworkflows.xyz/workflows/lead-analysis---personalized-email-generation-with-openai--firecrawl---gotohuman-8520


# Lead Analysis & Personalized Email Generation with OpenAI, Firecrawl & gotoHuman

### 1. Workflow Overview

This workflow automates lead analysis and personalized email outreach using AI and data integrations. It targets sales and marketing teams who want to quickly respond to new inbound leads by generating tailored outreach emails based on the lead’s company website, profile, and historical positive examples.

The workflow is logically divided into the following blocks:

- **1.1 Lead Reception & Email Validation:** Receives lead data via Typeform, extracts the email domain, and filters out personal email addresses (e.g., gmail.com).

- **1.2 Website Scraping & Summarization:** Scrapes the lead’s company website using Firecrawl and summarizes its content with OpenAI to understand the prospect’s business.

- **1.3 AI Lead Analysis & Email Drafting:** Uses an AI Sales Agent (Langchain + OpenAI GPT-4) to analyze the lead and generate a personalized outreach email, leveraging company profiles, ICP descriptions, and previously approved positive examples from gotoHuman.

- **1.4 Human Review & Approval:** Presents the AI-generated analysis and email draft in gotoHuman for human review and approval.

- **1.5 Sending Approved Email:** If approved, sends the personalized email through Gmail.

Supporting this flow are several utility and control nodes for branching, parsing, and error handling.

---

### 2. Block-by-Block Analysis

#### 2.1 Lead Reception & Email Validation

- **Overview:**  
  This block captures new lead submissions from a Typeform form, extracts the lead’s email domain to infer company website, and filters out personal email addresses to focus on business leads only.

- **Nodes Involved:**  
  - Typeform Trigger  
  - Extract Domain (Code)  
  - Flag personal email addresses (Code)  
  - Is Personal Email? (IF)

- **Node Details:**  

  **Typeform Trigger**  
  - Type: Trigger (Typeform Webhook)  
  - Role: Receives new lead form submissions from a specific Typeform form (ID: tcNgdBxH).  
  - Configuration: Linked to the Typeform form; no credentials set here but required for webhook.  
  - Input/Output: Triggered on new form submit, outputs lead data JSON.  
  - Edge Cases: Webhook failures, malformed form data.

  **Extract Domain**  
  - Type: Code (JavaScript)  
  - Role: Extracts the domain from the submitted email (e.g., "example.com") and constructs a website URL (https://example.com).  
  - Key Variables: `Work Email` field from the Typeform response.  
  - Output adds `domain` and `websiteUrl` fields to the item JSON.  
  - Edge Cases: Missing or malformed email, no '@' symbol, unusual email formats.

  **Flag personal email addresses**  
  - Type: Code (JavaScript)  
  - Role: Flags emails belonging to common personal providers (gmail, yahoo, outlook, etc.) by checking the domain’s main part.  
  - Uses a predefined list of common personal email domains.  
  - Adds boolean `isPersonalEmail` to JSON.  
  - Edge Cases: Domains not matching list but personal, false negatives or positives.

  **Is Personal Email?**  
  - Type: IF node  
  - Role: Branches workflow based on whether the email is personal.  
  - Condition: `$json.isPersonalEmail == true`  
  - True Branch: Terminates or no further processing.  
  - False Branch: Continues to website scraping.  
  - Edge Cases: Missing flag, type coercion errors.

---

#### 2.2 Website Scraping & Summarization

- **Overview:**  
  This block scrapes the lead’s company website for content and generates a concise markdown summary to inform the AI lead analysis.

- **Nodes Involved:**  
  - Scrape the website (Firecrawl)  
  - Summarize website (OpenAI)

- **Node Details:**  

  **Scrape the website**  
  - Type: Firecrawl node (web scraping)  
  - Role: Scrapes content from the URL extracted earlier (`websiteUrl`).  
  - Configuration: Operation “scrape,” no special request options.  
  - Inputs: URL from previous block.  
  - Outputs: Scraped website HTML/text content.  
  - Edge Cases: Website down, access denied, scraping blocked by robots.txt, empty content.

  **Summarize website**  
  - Type: OpenAI node (GPT-4.1-mini)  
  - Role: Summarizes scraped website content into 2-3 paragraphs in markdown, highlighting industry, positioning, products, recent activities.  
  - Configuration: Uses prompt with embedded scraped content, instructs summary format.  
  - Input: Scraped data JSON from Firecrawl.  
  - Output: Markdown summary text.  
  - Edge Cases: API timeouts, content too large, model hallucinations.

---

#### 2.3 AI Lead Analysis & Email Drafting

- **Overview:**  
  This block uses an AI Sales Agent to analyze the lead data, company summary, and internal documents to draft a personalized outreach email, referencing approved past examples to improve quality.

- **Nodes Involved:**  
  - AI Sales Agent (Langchain agent)  
  - OpenAI Chat Model (GPT-4.1-mini)  
  - Structured Output Parser (JSON schema)  
  - Get our company profile (Google Docs)  
  - Get our ICP description (Google Docs)  
  - Fetch approved history (gotoHuman HTTP Request)

- **Node Details:**  

  **AI Sales Agent**  
  - Type: Langchain Agent Node  
  - Role: Orchestrates AI reasoning and output generation. It uses the OpenAI model and several tools to fetch info and produce a structured response.  
  - Prompt: Includes lead info, website summary, today's date, and instructions to analyze and draft.  
  - Inputs: Website summary, company profile, ICP description, approved examples.  
  - Outputs: Structured JSON with fields like industry, reasoning, email draft subject/body.  
  - Edge Cases: Model errors, missing tool data, malformed output.

  **OpenAI Chat Model**  
  - Type: Language Model (GPT-4.1-mini)  
  - Role: Provides the underlying language generation for the AI Sales Agent.  
  - Configuration: GPT-4.1-mini selected for balance of power/cost.  
  - Inputs/Outputs: Connected as LM for AI Sales Agent.  
  - Edge Cases: API limits, latency, content filtering.

  **Structured Output Parser**  
  - Type: Langchain Output Parser  
  - Role: Validates and parses AI Sales Agent output against a strict JSON schema requiring industry, reasoning, budget assessment, priority rating, email draft, etc.  
  - Prevents malformed or incomplete AI responses.  
  - Edge Cases: Schema mismatches, missing required fields.

  **Get our company profile & Get our ICP description**  
  - Type: Google Docs Tool  
  - Role: Fetch company and Ideal Customer Profile descriptions from Google Docs to provide context to AI.  
  - Inputs: Document URLs (to be set).  
  - Outputs: Text content for AI input.  
  - Edge Cases: Credential errors, invalid URLs, API limits.

  **Fetch approved history**  
  - Type: HTTP Request (gotoHuman API)  
  - Role: Retrieves past human-approved lead analyses and email drafts as examples to guide AI generation.  
  - Configuration: Query parameters to fetch approved values only for specific fields.  
  - Authentication: Uses gotoHuman API credentials.  
  - Edge Cases: API errors, network failures, empty history.

---

#### 2.4 Human Review & Approval

- **Overview:**  
  The AI-generated lead analysis and email draft are sent to a human reviewer via the gotoHuman node, enabling edits and approval before sending.

- **Nodes Involved:**  
  - Wait for Human Approval (gotoHuman)  
  - Is approved? (IF)

- **Node Details:**  

  **Wait for Human Approval**  
  - Type: gotoHuman Review Node  
  - Role: Sends AI output with lead details and email draft to a pre-configured review template in gotoHuman for human validation and editing.  
  - Inputs: Lead email, company, website, AI-generated fields (budget, priority, reasoning, email subject/body), website summary.  
  - Configuration: Uses a specific gotoHuman review template ID to structure the review form.  
  - Outputs: Review response with approval status and possibly edited content.  
  - Edge Cases: No human response, timeout, review rejection.

  **Is approved?**  
  - Type: IF node  
  - Role: Checks if the human reviewer marked the lead as “approved.”  
  - Condition: Review response equals “approved.”  
  - True Branch: Proceeds to send email.  
  - False Branch: Stops workflow.  
  - Edge Cases: Unexpected review statuses.

---

#### 2.5 Sending Approved Email

- **Overview:**  
  After human approval, the personalized outreach email is sent via Gmail to the lead.

- **Nodes Involved:**  
  - Send a message (Gmail)

- **Node Details:**  

  **Send a message**  
  - Type: Gmail Node  
  - Role: Sends an email to the lead using the reviewed subject and body text.  
  - Inputs: Email address, subject, body from the gotoHuman review response.  
  - Configuration: Plain text email, attribution disabled.  
  - Credentials: Gmail OAuth2 credentials required.  
  - Edge Cases: Auth failures, SMTP errors, invalid email addresses.

---

### 3. Summary Table

| Node Name                  | Node Type                         | Functional Role                              | Input Node(s)              | Output Node(s)              | Sticky Note                                                                                                          |
|----------------------------|----------------------------------|----------------------------------------------|----------------------------|-----------------------------|---------------------------------------------------------------------------------------------------------------------|
| Typeform Trigger            | Typeform Trigger                 | Receives new lead form submissions           | -                          | Extract Domain              |                                                                                                                     |
| Extract Domain             | Code                            | Extracts email domain and constructs website URL | Typeform Trigger           | Flag personal email addresses | ## Receive lead and check email<br>We receive a new form submission incl. the email address and company name of the prospect and extract the website URL from the address. We proceed only for company email addresses. |
| Flag personal email addresses | Code                            | Flags emails from common personal providers  | Extract Domain             | Is Personal Email?           | ## Receive lead and check email<br>We receive a new form submission incl. the email address and company name of the prospect and extract the website URL from the address. We proceed only for company email addresses. |
| Is Personal Email?          | IF                              | Branches by personal email flag               | Flag personal email addresses | Scrape the website (No branch if true) | ## Receive lead and check email<br>We receive a new form submission incl. the email address and company name of the prospect and extract the website URL from the address. We proceed only for company email addresses. |
| Scrape the website          | Firecrawl                       | Scrapes lead’s company website                 | Is Personal Email? (False branch) | Summarize website           | ## Scrape and Summarize Company Website<br>We scrape the website using Firecrawl and summarize it with OpenAI      |
| Summarize website           | OpenAI (GPT-4.1-mini)           | Summarizes scraped website content             | Scrape the website          | AI Sales Agent              | ## Scrape and Summarize Company Website<br>We scrape the website using Firecrawl and summarize it with OpenAI      |
| Get our company profile     | Google Docs Tool                | Fetches company profile document content      | -                          | AI Sales Agent              | ## AI Analysis and Outreach Drafting<br>Our AI agent runs an analysis based on the lead information and documents describing our own company and the defined Ideal Customer Profiles. It also fetches previously approved examples from **gotoHuman** so you're effectively creating a **self-learning** agent. It responds with the analysis and the drafted outreach email. |
| Get our ICP description     | Google Docs Tool                | Fetches Ideal Customer Profile document content | -                         | AI Sales Agent              | ## AI Analysis and Outreach Drafting<br>Our AI agent runs an analysis based on the lead information and documents describing our own company and the defined Ideal Customer Profiles. It also fetches previously approved examples from **gotoHuman** so you're effectively creating a **self-learning** agent. It responds with the analysis and the drafted outreach email. |
| Fetch approved history      | HTTP Request (gotoHuman API)   | Retrieves past approved lead analyses/examples | -                          | AI Sales Agent              | ## AI Analysis and Outreach Drafting<br>Our AI agent runs an analysis based on the lead information and documents describing our own company and the defined Ideal Customer Profiles. It also fetches previously approved examples from **gotoHuman** so you're effectively creating a **self-learning** agent. It responds with the analysis and the drafted outreach email. |
| AI Sales Agent              | Langchain Agent                | Analyzes lead info and drafts personalized email | Summarize website, Get our company profile, Get our ICP description, Fetch approved history | Wait for Human Approval     | ## AI Analysis and Outreach Drafting<br>Our AI agent runs an analysis based on the lead information and documents describing our own company and the defined Ideal Customer Profiles. It also fetches previously approved examples from **gotoHuman** so you're effectively creating a **self-learning** agent. It responds with the analysis and the drafted outreach email. |
| Structured Output Parser    | Langchain Output Parser         | Validates and parses AI agent output          | AI Sales Agent (ai_outputParser) | AI Sales Agent              |                                                                                                                     |
| Wait for Human Approval     | gotoHuman                      | Presents AI output for human review and editing | AI Sales Agent             | Is approved?                | ![gotoHuman Logo](https://cdn1.gotohuman.com/public%2Fn8n-templates%2FgotoHuman%20full%20logo%201224px.png?alt=media)<br>## Send Human-Approved Email<br>We can now send our email including any edits made during the review and be sure that we are using high-quality content instead of AI slop. |
| Is approved?                | IF                             | Checks if email is approved by human reviewer | Wait for Human Approval    | Send a message              | ## Send Human-Approved Email<br>We can now send our email including any edits made during the review and be sure that we are using high-quality content instead of AI slop. |
| Send a message              | Gmail                          | Sends the approved personalized email         | Is approved?               | -                           | ## Send Human-Approved Email<br>We can now send our email including any edits made during the review and be sure that we are using high-quality content instead of AI slop. |
| Sticky Note                 | Sticky Note                    | Summary and setup instructions                  | -                          | -                           | ## 💼 Lead Outreach Agent<br>This AI workflow helps you quickly react to new leads with an initial personalized outreach. A great start of your lead nurturing sequence to avoid loosing precious leads that could turn into paying customers. <br>Most importantly it uses **gotoHuman** so you can review the AI-analysis and the AI-generated editable email draft before it is sent out in your name.<br><br>### How to set up<br>1. **Most importantly, [install the gotoHuman node](https://docs.gotohuman.com/Integrations/n8n#add-the-node) before importing this template!**<br>(Just add the node to a blank canvas before importing)<br>2. Set up your credentials for the different services<br>3. In gotoHuman, select and create the pre-built review template \"Lead Outreach Agent\" or import the ID: `T873fI1Xli5nt3eh33Rj`<br>4. Select this template in the gotoHuman node<br><br>## Requirements<br>You need accounts for<br>- gotoHuman (Human Supervision)<br>- OpenAI (AI Agent)<br>- Typeform (Lead Form Submissions)<br>- Firecrawl (Website Scraping)<br>- Gmail<br>- Google Docs (Company Wiki)<br><br>## How to customize<br>- Replace the Typeform trigger with any other way you might receive or find new leads<br>- Provide the _AI Sales Agent_ with more context to properly analyze the lead and create better personalized emails. Consider adding tools that allow the agent to fetch more infos about the prospect's company or personal profile, or to find out more about your specific product/service offerings and how your sales pitches look like. |
| Sticky Note1                | Sticky Note                    | Visual guide screenshots                        | -                          | -                           | ![pic1](https://cdn1.gotohuman.com/public%2Fn8n-templates%2Flead-outreach%2Freview1.jpg?alt=media)<br>![pic1](https://cdn1.gotohuman.com/public%2Fn8n-templates%2Flead-outreach%2Freview2.jpg?alt=media)<br>![pic1](https://cdn1.gotohuman.com/public%2Fn8n-templates%2Flead-outreach%2Freview3.jpg?alt=media) |
| Sticky Note2                | Sticky Note                    | Explanation for lead reception and email checking | -                          | -                           | ## Receive lead and check email<br>We receive a new form submission incl. the email address and company name of the prospect and extract the website URL from the address. We proceed only for company email addresses. |
| Sticky Note3                | Sticky Note                    | Explanation for scraping and summarization     | -                          | -                           | ## Scrape and Summarize Company Website<br>We scrape the website using Firecrawl and summarize it with OpenAI |
| Sticky Note4                | Sticky Note                    | Explanation for AI analysis and email drafting | -                          | -                           | ## AI Analysis and Outreach Drafting<br>Our AI agent runs an analysis based on the lead information and documents describing our own company and the defined Ideal Customer Profiles.<br>It also fetches previously approved examples from **gotoHuman** so you're effectively creating a **self-learning** agent.<br>It responds with the analysis and the drafted outreach email. |
| Sticky Note5                | Sticky Note                    | gotoHuman Logo                                  | -                          | -                           | ![gotoHuman Logo](https://cdn1.gotohuman.com/public%2Fn8n-templates%2FgotoHuman%20full%20logo%201224px.png?alt=media) |
| Sticky Note6                | Sticky Note                    | Explanation for sending human-approved email   | -                          | -                           | ## Send Human-Approved Email<br>We can now send our email including any edits made during the review and be sure that we are using high-quality content instead of AI slop. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Typeform Trigger Node**  
   - Type: Typeform Trigger  
   - Configuration: Set Form ID to your lead capture form (e.g., `tcNgdBxH`)  
   - Credentials: Connect your Typeform account  
   - Position: Start of workflow  

2. **Add a Code Node “Extract Domain”**  
   - Type: Code (JavaScript)  
   - Purpose: Extract domain from lead’s work email and create website URL  
   - Code snippet:  
     ```js
     const email = $input.item.json['Work Email'];
     const domain = email.split('@')[1];
     const websiteUrl = `https://${domain}`;
     $input.item.json.domain = domain;
     $input.item.json.websiteUrl = websiteUrl;
     return $input.item;
     ```  
   - Connect output of Typeform Trigger to this node  

3. **Add a Code Node “Flag personal email addresses”**  
   - Type: Code (JavaScript)  
   - Purpose: Flag if email domain is a common personal provider  
   - Code snippet:  
     ```js
     const commonProviders = ['gmail', 'yahoo', 'ymail', 'rocketmail', 'outlook', 'hotmail', 'live', 'msn', 'icloud', 'me', 'mac', 'aol', 'zoho', 'protonmail', 'mail', 'gmx'];
     const domain = $input.item.json.domain;
     const mainDomain = domain.split('.')[0].toLowerCase();
     $input.item.json.isPersonalEmail = commonProviders.includes(mainDomain);
     return $input.item;
     ```  
   - Connect output of “Extract Domain” to this node  

4. **Add an IF Node “Is Personal Email?”**  
   - Type: IF  
   - Condition: `$json.isPersonalEmail == true`  
   - Connect output of “Flag personal email addresses”  
   - True branch: No further action or end  
   - False branch: Continue workflow  

5. **Add Firecrawl Node “Scrape the website”**  
   - Type: Firecrawl (website scraping)  
   - Operation: Scrape  
   - URL: Use expression `{{$json.websiteUrl}}`  
   - Connect False branch of IF node to this node  

6. **Add OpenAI Node “Summarize website”**  
   - Type: OpenAI (GPT-4.1-mini)  
   - Model: GPT-4.1-mini  
   - Prompt: Summarize the scraped content in markdown with instructions on industry, products, strategic moves  
   - Input: Use scraped data content from Firecrawl node  
   - Connect output of Firecrawl node here  

7. **Add Google Docs Node “Get our company profile”**  
   - Type: Google Docs Tool  
   - Operation: Get document content  
   - Document URL: Set your company profile document URL  
   - Connect independently to AI Sales Agent (later)  

8. **Add Google Docs Node “Get our ICP description”**  
   - Type: Google Docs Tool  
   - Operation: Get document content  
   - Document URL: Set your Ideal Customer Profile document URL  
   - Connect independently to AI Sales Agent (later)  

9. **Add HTTP Request Node “Fetch approved history”**  
   - Type: HTTP Request  
   - URL: `https://api.gotohuman.com/queryResponses`  
   - Method: GET  
   - Query Parameters: formId, fieldIds (companySummary, reasoningInterest, reasoningIcp, personalizationHook, budget, priorityRating, emailToSendSubject, emailToSendBody), approvedValuesOnly=true  
   - Authentication: Use gotoHuman API credentials  
   - Connect independently to AI Sales Agent (later)  

10. **Add Langchain Agent Node “AI Sales Agent”**  
    - Type: Langchain Agent  
    - Provide prompt with lead info (email, company, website), website summary, today’s date  
    - Add tools: OpenAI Chat Model, Google Docs for company profile and ICP, gotoHuman approved history  
    - Connect outputs from Summarize Website, Get Company Profile, Get ICP Description, Fetch Approved History as tools inputs  
    - Connect output to Structured Output Parser  

11. **Add Langchain Output Parser Node “Structured Output Parser”**  
    - Type: Output Parser  
    - Schema: Define JSON schema requiring fields like industry, reasoning markdowns, budget, priority, email subject/body  
    - Connect output of this node back to AI Sales Agent’s ai_outputParser input  

12. **Add gotoHuman Node “Wait for Human Approval”**  
    - Type: gotoHuman Review Node  
    - Configure with your gotoHuman account and select/create review template “Lead Outreach Agent” (ID: `T873fI1Xli5nt3eh33Rj`)  
    - Map fields from AI Sales Agent output and lead info to gotoHuman fields (email, company, website, AI analysis, draft email, etc.)  
    - Connect output of AI Sales Agent to this node  

13. **Add IF Node “Is approved?”**  
    - Type: IF  
    - Condition: Check if gotoHuman review response equals “approved”  
    - Connect output of Wait for Human Approval node  

14. **Add Gmail Node “Send a message”**  
    - Type: Gmail  
    - Configure with Gmail OAuth2 credentials  
    - Set recipient to `{{$json.responseValues.leadEmail.value}}`  
    - Subject to `{{$json.responseValues.emailToSendSubject}}`  
    - Message body to `{{$json.responseValues.emailToSendBody.value}}`  
    - Plain text email, no attribution appended  
    - Connect True branch of “Is approved?” to this node  

15. **Add Sticky Notes**  
    - Add the provided sticky notes to document flow, setup instructions, branding, and visual guidance.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 | Context or Link                                                                                              |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------|
| Most importantly, [install the gotoHuman node](https://docs.gotohuman.com/Integrations/n8n#add-the-node) before importing this template to enable human review.                                                                                                                                                                                                                                                                                                                                                                                           | gotoHuman Integration Setup                                                                                   |
| The workflow requires accounts and credentials for: gotoHuman, OpenAI, Typeform, Firecrawl, Gmail, and Google Docs for document fetching.                                                                                                                                                                                                                                                                                                                                                                                                                  | Service Credential Requirements                                                                                |
| The AI Sales Agent benefits from additional context: company description, ICP, and previous good examples fetched dynamically from gotoHuman for a self-learning effect.                                                                                                                                                                                                                                                                                                                                                                                    | AI Context and Learning                                                                                         |
| Customization tips: Replace the Typeform trigger with other lead sources; enrich the AI agent with more contextual tools to improve lead analysis and email quality.                                                                                                                                                                                                                                                                                                                                                                                         | Workflow Customization Suggestions                                                                              |
| Visual references and screenshots for the gotoHuman review interface are included in sticky notes to guide user understanding.                                                                                                                                                                                                                                                                                                                                                                                                                             | Visual Guide Links<br>![pic1](https://cdn1.gotohuman.com/public%2Fn8n-templates%2Flead-outreach%2Freview1.jpg?alt=media) |
| The workflow uses GPT-4.1-mini model for a balance between cost and quality in summarization and email generation.                                                                                                                                                                                                                                                                                                                                                                                                                                          | AI Model Selection                                                                                              |

---

**Disclaimer:**  
The provided content originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled are legal and public.