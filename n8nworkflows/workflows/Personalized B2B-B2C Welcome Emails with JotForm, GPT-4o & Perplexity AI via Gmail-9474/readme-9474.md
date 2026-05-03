Personalized B2B/B2C Welcome Emails with JotForm, GPT-4o & Perplexity AI via Gmail

https://n8nworkflows.xyz/workflows/personalized-b2b-b2c-welcome-emails-with-jotform--gpt-4o---perplexity-ai-via-gmail-9474


# Personalized B2B/B2C Welcome Emails with JotForm, GPT-4o & Perplexity AI via Gmail

---

### 1. Workflow Overview

This workflow automates sending personalized welcome emails to new leads captured via a JotForm form. It intelligently distinguishes between business (B2B) and personal (B2C) email addresses to tailor the messaging accordingly. For business emails, it performs a company research step using Perplexity AI to create deeply personalized onboarding emails. For personal emails, it generates warm and friendly welcome messages without additional research. Both email variants are composed using OpenAI’s GPT-4o language model and then sent via Gmail.

The workflow’s logical blocks are:

- **1.1 Input Reception:** Trigger on new JotForm submissions capturing leads.
- **1.2 Email Domain Extraction & Classification:** Extract the email domain and determine if it is a work or personal email.
- **1.3 Company Research (B2B path):** If a work email, research the company with Perplexity AI.
- **1.4 AI Email Generation:** Use OpenAI GPT-4o to draft personalized emails based on the path:
    - B2B path includes company research content.
    - B2C path uses a simpler prompt without company data.
- **1.5 Email Dispatch:** Send the generated email via Gmail.
- **1.6 Workflow Guidance:** Sticky note providing comprehensive documentation and setup instructions.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  Listens for new lead submissions on a specific JotForm. Captures lead data including `name` and `email`.

- **Nodes Involved:**  
  - On New JotForm Submission

- **Node Details:**  
  - **On New JotForm Submission**  
    - Type: JotForm Trigger  
    - Configuration: Connected to a specific JotForm lead capture form; requires credentials for JotForm API.  
    - Key expressions: None at this point; captures the entire submission JSON.  
    - Inputs: External webhook trigger from JotForm on new submission.  
    - Outputs: Lead data including fields like `name` and `email`.  
    - Edge cases: Form field names must match exactly (`name`, `email`) or data extraction will fail. Webhook connectivity issues could cause missed triggers.

#### 2.2 Email Domain Extraction & Classification

- **Overview:**  
  Extracts the domain from the lead’s email to determine if it is a work or personal email, guiding the subsequent workflow path.

- **Nodes Involved:**  
  - Extract Domain from Email  
  - If Work Email

- **Node Details:**  
  - **Extract Domain from Email**  
    - Type: Set Node  
    - Configuration: Splits the lead’s email address at `@` to extract domain (e.g., `gmail.com`).  
    - Expression: `={{ $json.email.split('@')[1] }}`  
    - Input: Lead data from JotForm trigger node.  
    - Output: Original lead data plus added `domain_name` field.  
    - Edge cases: Email without `@` or malformed email may cause errors or empty domain.
  
  - **If Work Email**  
    - Type: If Node  
    - Configuration: Checks if extracted domain is NOT in a predefined array of personal email domains (`gmail.com`, `yahoo.com`, `outlook.com`, `yopmail.com`).  
    - Expression: Uses array `notContains` operator to classify domain as work email if domain is absent from personal domains list.  
    - Inputs: Output of domain extraction node.  
    - Outputs:  
      - True branch: Work email path  
      - False branch: Personal email path  
    - Edge cases: Domains not listed as personal but still personal (e.g., `icloud.com`) will be misclassified unless added. Case sensitivity and strict validation may cause misclassification if domain formatting varies.

#### 2.3 Company Research (B2B Path)

- **Overview:**  
  For work emails, performs company research using Perplexity AI based on the extracted domain to gather detailed company information to personalize the email.

- **Nodes Involved:**  
  - Research Company via Perplexity AI

- **Node Details:**  
  - **Research Company via Perplexity AI**  
    - Type: Perplexity AI Node  
    - Configuration: Uses `sonar-pro` model to query Perplexity AI with a prompt requesting detailed company overview based on `domain_name`.  
    - Key prompt includes: company name, industry, products, news, culture, achievements.  
    - Input: Work email branch from If node, includes `domain_name`.  
    - Output: JSON containing researched company data in `.message.content`.  
    - Edge cases: API rate limits, network errors, incomplete or outdated company data from Perplexity.  
    - Credentials: Requires Perplexity API account with access.

#### 2.4 AI Email Generation

- **Overview:**  
  Generates the personalized welcome email content using OpenAI's GPT-4o, leveraging company research for B2B leads or simpler prompts for B2C leads.

- **Nodes Involved:**  
  - AI Company Email Writer  
  - AI Personal Email Writer  
  - OpenAI Chat Model (used as language model node linked to AI Company Email Writer)  
  - OpenAI Chat Model1 (used as language model node linked to AI Personal Email Writer)

- **Node Details:**  
  - **AI Company Email Writer**  
    - Type: LangChain Agent (OpenAI) Node  
    - Configuration:  
      - Prompt includes lead info (`name`, `email`, `domain_name`), company research from Perplexity, and additional form data.  
      - System message instructs the model to write warm, professional, highly personalized B2B onboarding emails with actionable next steps, formatted as HTML with subject line at the top.  
    - Input: Output from Perplexity AI node and lead data.  
    - Output: HTML formatted email with subject line.  
    - Edge cases: Model response latency or API limits; prompt formatting errors could cause output inconsistencies.  
    - Credentials: OpenAI API credentials required.  
    - Version: Using GPT-4o model.

  - **AI Personal Email Writer**  
    - Type: LangChain Agent (OpenAI) Node  
    - Configuration:  
      - Prompt includes lead info without company research.  
      - System message similar but tailored to B2C style: warm, professional, personal welcome emails.  
      - Output format: HTML email with subject line.  
    - Input: Personal email branch from If node.  
    - Output: HTML formatted email with subject line.  
    - Edge cases: Same as above; simpler prompt reduces risk of missing data but still depends on API availability.  
    - Credentials: OpenAI API credentials required.  
    - Version: GPT-4o model.

  - **OpenAI Chat Model & OpenAI Chat Model1**  
    - Type: LangChain language model nodes for GPT-4o  
    - Role: Provide the language model backend for the Agent nodes.  
    - Configuration: Both set to use `gpt-4o` model.  
    - Input/Output: Connected as language model to respective Agent nodes.  
    - Edge cases: API key validity and rate limits must be monitored.

#### 2.5 Email Dispatch

- **Overview:**  
  Sends the generated personalized HTML email via Gmail to the lead’s email address.

- **Nodes Involved:**  
  - Merge  
  - Send Welcome Email

- **Node Details:**  
  - **Merge**  
    - Type: Merge Node  
    - Configuration: Combines output from both AI email writers (B2B and B2C paths) into a single stream for sending.  
    - Inputs:  
      - Input 1: From AI Company Email Writer  
      - Input 2: From AI Personal Email Writer  
    - Output: Unified data for email sending.  
    - Edge cases: Ensures only one path is active per execution; misconfiguration could cause duplicate or missing emails.

  - **Send Welcome Email**  
    - Type: Gmail Node  
    - Configuration:  
      - Sends email to the lead’s email from JotForm trigger.  
      - Subject line extracted dynamically from the generated email content or constructed using lead’s first name.  
      - Email body uses the generated HTML content.  
      - Gmail OAuth2 credentials required for authentication.  
      - Option to disable attribution for cleaner email.  
    - Input: Merged stream of final email content.  
    - Output: Confirmation of email sent.  
    - Edge cases: Gmail API quota limits, OAuth token expiration, invalid or blocked email addresses, HTML rendering issues.

#### 2.6 Workflow Guidance

- **Overview:**  
  Provides detailed inline documentation for users on the workflow’s purpose, setup, and customization options.

- **Nodes Involved:**  
  - Sticky Note

- **Node Details:**  
  - **Sticky Note**  
    - Type: Sticky Note Node  
    - Content: Extensive explanation of the workflow including use cases, step-by-step logic, setup instructions, and customization tips.  
    - Positioned outside main data flow purely for human reference.  
    - Edge cases: None functionally, but content must be kept updated with workflow changes.

---

### 3. Summary Table

| Node Name                    | Node Type                           | Functional Role                      | Input Node(s)                 | Output Node(s)                 | Sticky Note                                                                                                                           |
|------------------------------|-----------------------------------|------------------------------------|------------------------------|-------------------------------|---------------------------------------------------------------------------------------------------------------------------------------|
| On New JotForm Submission     | JotForm Trigger                   | Trigger on new lead submission     | -                            | Extract Domain from Email      | See Sticky Note for full workflow description and setup instructions.                                                                 |
| Extract Domain from Email     | Set                              | Extracts email domain               | On New JotForm Submission     | If Work Email                 | See Sticky Note for full workflow description and setup instructions.                                                                 |
| If Work Email                | If                               | Classify email as work or personal | Extract Domain from Email     | Research Company via Perplexity AI (True branch) / AI Personal Email Writer (False branch) | See Sticky Note for full workflow description and setup instructions.                                                                 |
| Research Company via Perplexity AI | Perplexity AI Node              | Perform company research            | If Work Email (True branch)   | AI Company Email Writer       | See Sticky Note for full workflow description and setup instructions.                                                                 |
| AI Company Email Writer       | LangChain Agent (OpenAI)          | Generate B2B personalized email    | Research Company via Perplexity AI | Merge                      | See Sticky Note for full workflow description and setup instructions.                                                                 |
| AI Personal Email Writer      | LangChain Agent (OpenAI)          | Generate B2C personalized email    | If Work Email (False branch)  | Merge                        | See Sticky Note for full workflow description and setup instructions.                                                                 |
| OpenAI Chat Model             | LangChain LM Chat OpenAI          | Language model for B2B email writer| -                            | AI Company Email Writer (lmChatOpenAi input) | See Sticky Note for full workflow description and setup instructions.                                                                 |
| OpenAI Chat Model1            | LangChain LM Chat OpenAI          | Language model for B2C email writer| -                            | AI Personal Email Writer (lmChatOpenAi input) | See Sticky Note for full workflow description and setup instructions.                                                                 |
| Merge                        | Merge                            | Combine email outputs from both paths | AI Company Email Writer, AI Personal Email Writer | Send Welcome Email           | See Sticky Note for full workflow description and setup instructions.                                                                 |
| Send Welcome Email            | Gmail                           | Send final email to lead           | Merge                        | -                             | See Sticky Note for full workflow description and setup instructions.                                                                 |
| Sticky Note                  | Sticky Note                     | Documentation and instructions     | -                            | -                             | ## Send smart, personalized welcome emails to any JotForm lead... [Full content in node]                                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create JotForm Trigger Node**  
   - Node Type: JotForm Trigger  
   - Configure credentials for JotForm API.  
   - Select the specific lead capture form.  
   - Ensure form fields include `name` and `email`.

2. **Create Set Node to Extract Domain**  
   - Node Type: Set  
   - Add a string field named `domain_name`.  
   - Set its value using expression: `{{$json.email.split('@')[1]}}`  
   - Enable "Keep Only Set" disabled to preserve all other fields.

3. **Create If Node to Check Email Type**  
   - Node Type: If  
   - Condition: Check if `domain_name` is NOT in `[ "gmail.com", "yahoo.com", "outlook.com", "yopmail.com" ]` array.  
   - Use `notContains` operator with strict type validation and case sensitivity as configured.

4. **Create Perplexity AI Node for Company Research**  
   - Node Type: Perplexity AI  
   - Authenticate with Perplexity API credentials.  
   - Model: `sonar-pro`  
   - Prompt:  
     ```
     Research the company with domain {{ $json.domain_name }}. Provide a comprehensive overview including:

     1. Company name and overview
     2. Industry and market position
     3. Key products or services
     4. Recent news or developments
     5. Company culture and values
     6. Any notable achievements or recognition

     Provide detailed, accurate information that would help personalize a welcome email.
     ```
   - Connect input from If node True branch.

5. **Create LangChain OpenAI Chat Model Node for B2B**  
   - Node Type: LangChain LM Chat OpenAI  
   - Model: `gpt-4o`  
   - Authenticate with OpenAI API credentials.

6. **Create LangChain Agent Node for B2B Email Writer**  
   - Node Type: LangChain Agent  
   - Connect language model input from above.  
   - Prompt:  
     ```
     Create a highly personalized welcome onboarding email for:

     Lead Information:
     - Name: {{ $('On New JotForm Submission').item.json.name }}
     - Email: {{ $('On New JotForm Submission').item.json.email }}
     - Company Domain: {{ $('Extract Domain from Email').item.json.domain_name }}

     Company Research:
     {{ $('Research Company via Perplexity AI').item.json.message.content }}

     Additional Form Data:
     {{ JSON.stringify($('On New JotForm Submission').item.json, null, 2) }}

     ---
     Create a warm, professional welcome email that:
     1. Addresses them by name
     2. Shows we've researched their company
     3. Demonstrates understanding of their industry/challenges
     4. Provides clear next steps
     5. Makes them excited to get started
     6. Includes a compelling subject line

     Format the output as HTML email content with a subject line at the top.
     ```
   - System message:  
     ```
     You are an expert email copywriter specializing in personalized B2B onboarding communications. Your emails are:

     - Warm yet professional
     - Highly personalized based on company research
     - Engaging and conversational
     - Clear about next steps
     - Focused on value and relevance to their business

     Create emails that make recipients feel valued and understood. Use the company research to demonstrate genuine interest in their business. Keep the tone friendly but professional, and always include actionable next steps.

     Only output the response in HTML and nothing else.
     ```

7. **Create LangChain OpenAI Chat Model Node for B2C**  
   - Node Type: LangChain LM Chat OpenAI  
   - Model: `gpt-4o`  
   - Authenticate with OpenAI API credentials.

8. **Create LangChain Agent Node for Personal Email Writer**  
   - Node Type: LangChain Agent  
   - Connect language model input from above.  
   - Prompt:  
     ```
     Create a highly personalized welcome onboarding email for:

     Lead Information:
     - Name: {{ $('On New JotForm Submission').item.json.name }}
     - Email: {{ $('On New JotForm Submission').item.json.email }}

     Additional Form Data:
     {{ JSON.stringify($('On New JotForm Submission').item.json, null, 2) }}

     ---
     Create a warm, professional personal welcome email that:
     1. Addresses them by name
     2. Demonstrates understanding of their challenges
     3. Provides clear next steps
     4. Makes them excited to get started
     5. Includes a compelling subject line

     Format the output as HTML email content with a subject line at the top.
     ```
   - System message: Same as B2B system message but adapted for personal emails.

9. **Create Merge Node**  
   - Node Type: Merge  
   - Inputs:  
     - Input 1: Output from B2B Email Writer node  
     - Input 2: Output from B2C Email Writer node  
   - Use default merge mode.

10. **Create Gmail Node to Send Email**  
    - Node Type: Gmail  
    - Authenticate with Gmail OAuth2 credentials.  
    - Set recipient email dynamically: `={{ $('On New JotForm Submission').item.json.email }}`  
    - Set email subject dynamically using expression to extract first name from lead name, e.g., `"Welcome to Our Platform, {{ $('On New JotForm Submission').item.json.name.split(' ')[0] }}!"`  
    - Set email body to the generated HTML output: `={{ $json.output }}`  
    - Disable attribution for cleaner email.

11. **Connect Nodes in Order**  
    - On New JotForm Submission → Extract Domain from Email → If Work Email  
    - If Work Email True → Research Company via Perplexity AI → AI Company Email Writer (with OpenAI Chat Model) → Merge  
    - If Work Email False → AI Personal Email Writer (with OpenAI Chat Model1) → Merge  
    - Merge → Send Welcome Email

12. **Add Sticky Note** (Optional)  
    - Add a sticky note node anywhere in the canvas with workflow documentation and setup instructions for team reference.

13. **Activate Workflow**  
    - Ensure all credentials are valid and authorized.  
    - Turn on the workflow to start processing new JotForm submissions automatically.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 | Context or Link                                                                                       |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| This workflow intelligently qualifies new JotForm leads and sends the perfect welcome email every time. It detects whether a lead is using a business or personal email address and tailors the outreach accordingly—either with deep company research for B2B leads or a warm, direct welcome for B2C leads.                                                                                                                                                                                                                         | Workflow purpose and logic overview (Sticky Note content)                                           |
| Your JotForm must contain fields named exactly `name` and `email` for proper data extraction. Additional fields (e.g., `company_size`) can be included to enhance personalization.                                                                                                                                                                                                                                                                                                                                          | JotForm configuration requirements                                                                  |
| Add more personal email domains to the `If Work Email` node’s array (e.g., `icloud.com`) to improve email classification accuracy.                                                                                                                                                                                                                                                                                                                                                                                         | Customization tip                                                                                   |
| Requires API access and valid credentials for Perplexity AI, OpenAI, JotForm, and Gmail OAuth2.                                                                                                                                                                                                                                                                                                                                                                                                                               | Credential requirements                                                                             |
| OpenAI’s GPT-4o model is used for email generation, providing advanced language understanding for personalized content creation.                                                                                                                                                                                                                                                                                                                                                                                             | Technical detail about AI model                                                                     |
| Gmail node’s “Append Attribution” option is disabled to ensure the email appears professional and free of automation footers.                                                                                                                                                                                                                                                                                                                                                                                                 | Email sending configuration detail                                                                 |
| For more information on n8n integrations: [n8n Documentation](https://docs.n8n.io/integrations/)                                                                                                                                                                                                                                                                                                                                                                                                                               | n8n official documentation                                                                          |

---

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

---