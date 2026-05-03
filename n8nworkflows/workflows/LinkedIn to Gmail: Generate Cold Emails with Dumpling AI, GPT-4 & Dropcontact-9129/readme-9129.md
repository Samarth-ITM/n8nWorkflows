LinkedIn to Gmail: Generate Cold Emails with Dumpling AI, GPT-4 & Dropcontact

https://n8nworkflows.xyz/workflows/linkedin-to-gmail--generate-cold-emails-with-dumpling-ai--gpt-4---dropcontact-9129


# LinkedIn to Gmail: Generate Cold Emails with Dumpling AI, GPT-4 & Dropcontact

---

### 1. Workflow Overview

This workflow automates the generation and sending of personalized cold emails based on submitted LinkedIn company profiles. It targets digital marketing agencies or sales teams aiming to streamline lead generation and outreach by leveraging AI and data enrichment services. The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Captures LinkedIn profile URLs submitted via a web form trigger.
- **1.2 Company Data Extraction:** Uses Dumpling AI to extract detailed company information from the LinkedIn profile URL.
- **1.3 Contact Enrichment:** Enhances company data with contact details like names and emails using Dropcontact.
- **1.4 Cold Email Generation:** Utilizes GPT-4 via OpenAI to craft a personalized cold email and subject line based on enriched contact and company data.
- **1.5 Email Sending:** Sends the generated cold email through Gmail using OAuth2 credentials.
- **1.6 Lead Logging:** Records the lead details into an Airtable base for tracking and future reference.

A sticky note summarizes the workflow purpose and requirements.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
This block serves as the entry point where users submit a LinkedIn company profile URL via a web form, triggering the workflow.

**Nodes Involved:**  
- Trigger on LinkedIn Profile Submission

**Node Details:**

- **Trigger on LinkedIn Profile Submission**  
  - **Type:** Form Trigger (Webhook-based)  
  - **Role:** Collects LinkedIn company profile URL submitted via a web form titled "Linkedin profile".  
  - **Configuration:** One form field named "profile URL" for URL input. Uses a webhook for receiving submissions.  
  - **Inputs:** External form submission HTTP request.  
  - **Outputs:** Outputs JSON with the submitted profile URL for downstream processing.  
  - **Edge Cases:** Missing or malformed URL submissions may cause failures downstream; no explicit validation here. Webhook availability and permissions must be ensured for external form calls.  
  - **Version Specifics:** Uses n8n FormTrigger node version 2.2.

#### 2.2 Company Data Extraction

**Overview:**  
Takes the submitted LinkedIn profile URL and calls Dumpling AI's API to extract detailed company information including name, description, industry, website, employee count, posts, and location.

**Nodes Involved:**  
- Extract Company Info (Dumpling AI)

**Node Details:**

- **Extract Company Info (Dumpling AI)**  
  - **Type:** HTTP Request  
  - **Role:** Sends POST request to Dumpling AI API endpoint for LinkedIn company data extraction.  
  - **Configuration:**  
    - URL: `https://app.dumplingai.com/api/v1/linkedin/company`  
    - Method: POST  
    - Body: JSON containing the submitted profile URL from previous node (`{{$json['profile URL']}}`).  
    - Authentication: HTTP header auth with Dumpling AI API key credential.  
  - **Inputs:** Receives JSON with LinkedIn profile URL from the form trigger.  
  - **Outputs:** JSON with company data (name, website, description, employees, location, posts, etc.).  
  - **Edge Cases:**  
    - API errors due to invalid URL or rate limits.  
    - Network timeouts or authentication failures.  
    - Incomplete or missing company data in response.  
  - **Version:** HTTP Request node v4.2.

#### 2.3 Contact Enrichment

**Overview:**  
Uses Dropcontact service to enrich the company data with contact details such as full name, email, location, and website for lead personalization.

**Nodes Involved:**  
- Enrich Contact (Dropcontact)

**Node Details:**

- **Enrich Contact (Dropcontact)**  
  - **Type:** Dropcontact node (specialized integration)  
  - **Role:** Enhances company data to find valid contact information.  
  - **Configuration:**  
    - Wait time: 60 seconds (to ensure API processing).  
    - Additional fields mapped from Dumpling AI output: country, website, LinkedIn URL, full name.  
    - Credentials: Dropcontact API key.  
  - **Inputs:** Company data JSON from Dumpling AI node output.  
  - **Outputs:** Enriched contact information including emails, phone, civility, full name, etc.  
  - **Edge Cases:**  
    - API rate limits or quota exhaustion.  
    - Missing or ambiguous contact info.  
    - Delay in processing due to wait time.  
  - **Version:** Dropcontact node v1.

#### 2.4 Cold Email Generation

**Overview:**  
Generates a personalized cold email and subject line using OpenAI's GPT-4, based on enriched contact and company description data.

**Nodes Involved:**  
- Generate Cold Email (GPT-4)

**Node Details:**

- **Generate Cold Email (GPT-4)**  
  - **Type:** LangChain OpenAI node (@n8n/n8n-nodes-langchain.openAi)  
  - **Role:** Creates a JSON-formatted email subject and HTML body tailored to the lead.  
  - **Configuration:**  
    - Model: GPT-4 (ID "gpt-4.1").  
    - Messages: System prompt defines style and structure of email (professional, warm, personalized, max 120 words, includes signature).  
    - Input variables: Lead name (`$json.full_name`) and company description (from Dumpling AI data).  
    - Output: JSON with `"subject"` and `"email"` fields.  
  - **Inputs:** Enriched contact data from Dropcontact node.  
  - **Outputs:** JSON object containing email subject and HTML email body.  
  - **Edge Cases:**  
    - API authentication or rate limit errors.  
    - Failure to parse expressions for inputs.  
    - Unexpected output formatting if prompt or API changes.  
  - **Version:** LangChain OpenAI node v1.8.

#### 2.5 Email Sending

**Overview:**  
Sends the generated cold email via Gmail using OAuth2 authentication.

**Nodes Involved:**  
- Send Cold Email via Gmail

**Node Details:**

- **Send Cold Email via Gmail**  
  - **Type:** Gmail node  
  - **Role:** Sends email with personalized subject and message content.  
  - **Configuration:**  
    - Recipient email hardcoded as `example@gmail.com` (should be parameterized for production).  
    - Subject and message content pulled from GPT-4 output (`$json.message.content.subject` and `.email`).  
    - Gmail OAuth2 credentials configured.  
    - Option to disable attribution append set to false (no attribution line added).  
  - **Inputs:** JSON with email content from GPT-4 node.  
  - **Outputs:** Gmail API response confirming message sent.  
  - **Edge Cases:**  
    - OAuth token expiry or permission errors.  
    - Invalid recipient email (currently hardcoded).  
    - Gmail API rate limits or quota errors.  
  - **Version:** Gmail node v2.1.

#### 2.6 Lead Logging

**Overview:**  
Records the extracted company and lead data into an Airtable base for centralized lead management.

**Nodes Involved:**  
- Log Lead to Airtable

**Node Details:**

- **Log Lead to Airtable**  
  - **Type:** Airtable node  
  - **Role:** Creates a new record in specified Airtable base and table with lead details.  
  - **Configuration:**  
    - Base and Table selected from a predefined list (linked to Airtable URLs).  
    - Fields mapped include company name, employee count, website, LinkedIn URL, and company description (from Dumpling AI output).  
    - Uses Airtable Personal Access Token credentials.  
  - **Inputs:** JSON with company data from Gmail node output (passed through).  
  - **Outputs:** Confirmation of record creation in Airtable.  
  - **Edge Cases:**  
    - API key or permissions issues.  
    - Schema mismatch or missing required fields.  
    - Rate limits or network errors.  
  - **Version:** Airtable node v2.1.

#### 2.7 Documentation Sticky Note

**Overview:**  
Provides a summarized description and instructions for the workflow.

**Nodes Involved:**  
- Sticky Note

**Node Details:**

- **Sticky Note**  
  - **Type:** Sticky Note node  
  - **Role:** Describes workflow steps, requirements, and outputs for user clarity.  
  - **Configuration:** Content details the 6-step workflow process and required credentials.  
  - **Inputs/Outputs:** None (informational).  
  - **Edge Cases:** N/A.

---

### 3. Summary Table

| Node Name                         | Node Type                        | Functional Role                       | Input Node(s)                      | Output Node(s)                | Sticky Note                                                                                              |
|----------------------------------|---------------------------------|-------------------------------------|----------------------------------|------------------------------|--------------------------------------------------------------------------------------------------------|
| Trigger on LinkedIn Profile Submission | Form Trigger (Webhook)           | Input reception of LinkedIn URL     | None                             | Extract Company Info (Dumpling AI) | Describes overall workflow steps and requirements (shared sticky note covers multiple nodes).          |
| Extract Company Info (Dumpling AI) | HTTP Request                    | Extract company info from LinkedIn  | Trigger on LinkedIn Profile Submission | Enrich Contact (Dropcontact)  | See above                                                                                              |
| Enrich Contact (Dropcontact)       | Dropcontact node                | Enrich contact with name/email data | Extract Company Info (Dumpling AI) | Generate Cold Email (GPT-4)    | See above                                                                                              |
| Generate Cold Email (GPT-4)         | LangChain OpenAI node           | Generate personalized cold email    | Enrich Contact (Dropcontact)     | Send Cold Email via Gmail      | See above                                                                                              |
| Send Cold Email via Gmail           | Gmail                          | Send email to lead                  | Generate Cold Email (GPT-4)       | Log Lead to Airtable           | See above                                                                                              |
| Log Lead to Airtable                | Airtable                       | Log lead details                    | Send Cold Email via Gmail         | None                         | See above                                                                                              |
| Sticky Note                        | Sticky Note                    | Documentation                      | None                             | None                         | Summarizes workflow, key steps, and credential requirements for Dumpling AI, Dropcontact, OpenAI, Gmail, Airtable |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node: "Trigger on LinkedIn Profile Submission"**  
   - Type: Form Trigger  
   - Configure form title: "Linkedin profile"  
   - Add one form field: Label "profile URL" (for LinkedIn company profile URL input)  
   - Note webhook URL generated will receive submissions.

2. **Add HTTP Request Node: "Extract Company Info (Dumpling AI)"**  
   - Connect from Form Trigger node output.  
   - Method: POST  
   - URL: `https://app.dumplingai.com/api/v1/linkedin/company`  
   - Body type: JSON  
   - Body content: `{ "url": "{{ $json['profile URL'] }}" }` using expression to pull submitted URL.  
   - Authentication: HTTP Header Auth with Dumpling AI API key credential configured.  
   - Node version: 4.2 or later.

3. **Add Dropcontact Node: "Enrich Contact (Dropcontact)"**  
   - Connect from Dumpling AI node output.  
   - Configure wait time: 60 seconds.  
   - Map additional fields from Dumpling AI output using expressions:  
     - country: `{{$json.location.country}}`  
     - website: `{{$json.website}}`  
     - linkedin: `{{$json.url}}`  
     - full_name: `{{$json.name}}`  
   - Use Dropcontact API credentials.  
   - Node version: 1.

4. **Add LangChain OpenAI Node: "Generate Cold Email (GPT-4)"**  
   - Connect from Dropcontact node output.  
   - Model: Select GPT-4 (model ID `"gpt-4.1"`).  
   - Set messages:  
     - System message containing detailed prompt instructions for writing a personalized cold email and subject line, requesting strict JSON output with "subject" and "email" keys.  
     - Pass input fields via expressions:  
       - Lead name: `{{$json.full_name}}`  
       - Company description from Dumpling AI output: `{{ $('Extract Company Info (Dumpling AI)').item.json.description }}`  
   - Enable JSON output parsing.  
   - Use OpenAI API credentials.  
   - Node version: 1.8.

5. **Add Gmail Node: "Send Cold Email via Gmail"**  
   - Connect from GPT-4 node output.  
   - Set recipient email: currently hardcoded as `"example@gmail.com"` (replace with dynamic email from Dropcontact if desired).  
   - Subject: `{{$json.message.content.subject}}` from GPT-4 output.  
   - Message body: `{{$json.message.content.email}}` (HTML content).  
   - Disable attribution append.  
   - Use Gmail OAuth2 credentials.  
   - Node version: 2.1.

6. **Add Airtable Node: "Log Lead to Airtable"**  
   - Connect from Gmail node output.  
   - Choose Airtable base and table for leads logging.  
   - Map fields with expressions from Dumpling AI node output:  
     - Name: `{{ $('Extract Company Info (Dumpling AI)').item.json.name }}`  
     - People (employee count): `{{ $('Extract Company Info (Dumpling AI)').item.json.employeeCount }}`  
     - Website: `{{ $('Extract Company Info (Dumpling AI)').item.json.website }}`  
     - LinkedIn Company URL: `{{ $('Extract Company Info (Dumpling AI)').item.json.url }}`  
     - Company Analysis (description): `{{ $('Extract Company Info (Dumpling AI)').item.json.description }}`  
   - Use Airtable Personal Access Token credentials.  
   - Node version: 2.1.

7. **Optional: Add Sticky Note Node for Documentation**  
   - Paste workflow summary including:  
     - Step-by-step process  
     - Required credentials (Dumpling AI, Dropcontact, OpenAI, Gmail, Airtable)  
     - Output expectations.

8. **Connect Nodes in Sequence:**  
   Trigger → Dumpling AI → Dropcontact → GPT-4 → Gmail → Airtable

9. **Credentials Setup:**  
   - Dumpling AI: HTTP Header Auth with API key.  
   - Dropcontact: API key credential.  
   - OpenAI: OpenAI API key credential.  
   - Gmail: OAuth2 credential with send mail scope.  
   - Airtable: Personal Access Token with write access.

10. **Testing and Validation:**  
    - Test form submission with valid LinkedIn company profile URL.  
    - Verify API responses at each step.  
    - Validate email content correctness and format.  
    - Confirm email delivery and Airtable record creation.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                | Context or Link                                                                                  |
|---------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| This workflow leverages Dumpling AI, Dropcontact, OpenAI GPT-4, Gmail, and Airtable APIs for seamless cold email generation and sending.  | Requires API keys and OAuth2 credentials for each service.                                     |
| The cold email generation prompt is carefully crafted to produce professional, warm, and personalized emails under 120 words in HTML.      | Embedded in GPT-4 node system message configuration.                                           |
| Gmail recipient email is currently hardcoded as "example@gmail.com" and should be adapted for real lead emails.                             | Modify the "Send Cold Email via Gmail" node to use dynamic email addresses from enriched data. |
| Dumpling AI enriches company data including LinkedIn posts and employee details, useful for detailed personalization beyond this workflow. | See Dumpling AI documentation: https://app.dumplingai.com/docs                               |
| Dropcontact enriches contacts with validated emails and phone numbers, improving deliverability and personalization.                       | See Dropcontact API docs: https://dropcontact.com/en/api                                       |
| Airtable base and table must be pre-configured with matching schema columns to log leads correctly.                                          | https://airtable.com/                                                                          |
| Workflow designed for n8n version supporting nodes FormTrigger v2.2, HTTP Request v4.2, Dropcontact v1, LangChain OpenAI v1.8, Gmail v2.1, Airtable v2.1 | Node versions compatibility important for stable operation.                                   |

---

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

---