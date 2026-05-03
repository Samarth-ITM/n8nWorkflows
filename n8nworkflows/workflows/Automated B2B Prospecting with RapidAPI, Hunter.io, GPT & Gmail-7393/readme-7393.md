Automated B2B Prospecting with RapidAPI, Hunter.io, GPT & Gmail

https://n8nworkflows.xyz/workflows/automated-b2b-prospecting-with-rapidapi--hunter-io--gpt---gmail-7393


# Automated B2B Prospecting with RapidAPI, Hunter.io, GPT & Gmail

### 1. Workflow Overview

This workflow automates B2B prospecting by searching for local businesses based on user input, extracting relevant company and contact information, generating personalized outreach emails using AI, and sending these emails via Gmail. It also creates Google Tasks for manual follow-ups when automated outreach is not possible.

Logical blocks include:

- **1.1 Input Reception:** Form-triggered node to capture prospecting criteria.
- **1.2 Business Search & Filtering:** Query local business data via RapidAPI, split and filter results.
- **1.3 Website Content Scraping:** Scrape business websites for title and description metadata.
- **1.4 Contact Discovery:** Find and validate contacts using Hunter.io.
- **1.5 AI-Driven Email Generation:** Generate personalized email bodies using OpenAI GPT agent.
- **1.6 Email Assembly & Dispatch:** Format email content and send via Gmail.
- **1.7 Manual Follow-up Task Creation:** Create Google Tasks for leads without websites.
- **1.8 Documentation & Credentials Setup:** Sticky notes guide users on setup and customization.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:** Receives user input specifying business sector, city, and number of leads via a web form.
- **Nodes Involved:** `compagny_search`
- **Node Details:**
  - **Type:** Form Trigger
  - **Role:** Initiates workflow on form submission.
  - **Configuration:** Form fields for "Business Sector" (required), "city" (optional), and "Number of Leads" (optional).
  - **Expressions:** Inputs are accessed as `$json["Business Sector"]`, `$json.city`, `$json["Number of Leads"]`.
  - **Connections:** Outputs to `Search Local Businesses`.
  - **Failure Modes:** Form submission errors, missing required fields.
  - **Version:** 2.2

#### 1.2 Business Search & Filtering

- **Overview:** Queries the RapidAPI Local Business Data API to retrieve businesses matching criteria; splits list into individual businesses.
- **Nodes Involved:** `Search Local Businesses`, `Split Out Businesses`, `If Website Exists`
- **Node Details:**

  - `Search Local Businesses`:
    - **Type:** HTTP Request
    - **Role:** Calls RapidAPI endpoint with query parameters (business sector + city, limit, zoom, language, region).
    - **Authentication:** HTTP Header Auth with RapidAPI key credential.
    - **Expressions:** Query parameters use form inputs e.g., `={{ $json["Business Sector"] }} {{ $json.city }}`.
    - **Failure Modes:** API errors (auth failure, rate limits, network issues).
  
  - `Split Out Businesses`:
    - **Type:** Split Out
    - **Role:** Splits the array of businesses in `data` field into individual items for separate processing.
    - **Failure Modes:** Empty or malformed `data` field.
  
  - `If Website Exists`:
    - **Type:** If
    - **Role:** Checks if the `website` field exists and is non-empty.
    - **Condition:** String not empty check on `$json.website`.
    - **Output:** If true, proceed to scrape website; else to manual task creation.
    - **Failure Modes:** Missing or malformed website URLs.

- **Connections:**
  - `Search Local Businesses` → `Split Out Businesses`
  - `Split Out Businesses` → `If Website Exists`
  - `If Website Exists` True → `Scrape Website`
  - `If Website Exists` False → `Set Task Data`

#### 1.3 Website Content Scraping

- **Overview:** Attempts to scrape website content, then extracts title and meta description if scraping succeeds.
- **Nodes Involved:** `Scrape Website`, `If Scraping Succeeded`, `Extract Title & Description`
- **Node Details:**

  - `Scrape Website`:
    - **Type:** HTTP Request
    - **Role:** Performs a GET request to the business website URL.
    - **Error Handling:** Continues workflow on error (e.g., site unreachable).
    - **Failure Modes:** Timeout, 404 errors, SSL issues.
  
  - `If Scraping Succeeded`:
    - **Type:** If
    - **Role:** Checks if scraping returned an error property.
    - **Condition:** True if no `error` property in JSON.
    - **Output:** True branch leads to content extraction node.
  
  - `Extract Title & Description`:
    - **Type:** HTML Extraction Node
    - **Role:** Extracts `<title>` text and meta description content from HTML.
    - **Selectors:** `title` tag and `meta[name="description"]`.
    - **Failure Modes:** Missing tags, malformed HTML.

- **Connections:**
  - `If Website Exists` True → `Scrape Website`
  - `Scrape Website` → `If Scraping Succeeded`
  - `If Scraping Succeeded` True → `Extract Title & Description`
  - `Extract Title & Description` → `Find Contacts (Hunter)`

#### 1.4 Contact Discovery

- **Overview:** Uses Hunter.io to find contacts associated with the company domain, then filters valid contacts.
- **Nodes Involved:** `Find Contacts (Hunter)`, `Split Out Contacts`, `If Contact Is Valid`
- **Node Details:**

  - `Find Contacts (Hunter)`:
    - **Type:** Hunter.io API Node
    - **Role:** Searches contacts by domain extracted from business website TLD.
    - **Limit:** Up to 10 contacts.
    - **Authentication:** Hunter.io API credentials required.
    - **Failure Modes:** API quota exceeded, invalid domain, connectivity issues.
  
  - `Split Out Contacts`:
    - **Type:** Split Out
    - **Role:** Splits contacts array into individual items.
  
  - `If Contact Is Valid`:
    - **Type:** If
    - **Role:** Filters contacts where first name is not empty and verification status is "valid".
    - **Condition:** Checks `$json.first_name` not empty and Hunter verification equals "valid".
    - **Output:** Valid contacts proceed to AI email generation.

- **Connections:**
  - `Extract Title & Description` → `Find Contacts (Hunter)`
  - `Find Contacts (Hunter)` → `Split Out Contacts`
  - `Split Out Contacts` → `If Contact Is Valid`
  - `If Contact Is Valid` True → `Generate Email Body (AI)`

#### 1.5 AI-Driven Email Generation

- **Overview:** Generates personalized email bodies for each valid contact using OpenAI GPT agent with a sales prospecting prompt.
- **Nodes Involved:** `Generate Email Body (AI)`, `OpenAI Chat Model`, `Format AI Output`
- **Node Details:**

  - `OpenAI Chat Model`:
    - **Type:** Langchain OpenAI Chat Node
    - **Role:** Provides GPT-5 nano-2025-08-07 model access.
    - **Credential:** OpenAI API key required.
  
  - `Generate Email Body (AI)`:
    - **Type:** Langchain Agent Node
    - **Role:** Sends prompt text describing contact and company info to generate short sales outreach email.
    - **Prompt Includes:** Contact first name, job title, company name, website title and description.
    - **System Message:** Specifies tone and structure of email (friendly, professional, personalized).
    - **Failure Modes:** API rate limits, prompt errors.
  
  - `Format AI Output`:
    - **Type:** Code Node
    - **Role:** Formats raw AI output text into readable paragraphs by splitting on sentence endings.
    - **Code:** JavaScript that cleans and structures the AI response.
    - **Failure Modes:** Unexpected AI output format.

- **Connections:**
  - `If Contact Is Valid` True → `Generate Email Body (AI)`
  - `OpenAI Chat Model` → `Generate Email Body (AI)` (AI model linkage)
  - `Generate Email Body (AI)` → `Format AI Output`
  - `Format AI Output` → `Assemble Final Email`

#### 1.6 Email Assembly & Dispatch

- **Overview:** Constructs email subject and body using AI output and sends email via Gmail.
- **Nodes Involved:** `Assemble Final Email`, `Send Email (Gmail)`
- **Node Details:**

  - `Assemble Final Email`:
    - **Type:** Set Node
    - **Role:** Creates `email_subject` and `email_body` strings with dynamic values.
    - **Subject:** "Developing the business of [Company Name]"
    - **Body:** Personalized greeting with AI-generated email text and closing.
  
  - `Send Email (Gmail)`:
    - **Type:** Gmail Node
    - **Role:** Sends the email to the contact's email address found by Hunter.io.
    - **Credentials:** Gmail OAuth2 credentials required.
    - **Failure Modes:** Authentication failure, quota limits, invalid email addresses.

- **Connections:**
  - `Format AI Output` → `Assemble Final Email`
  - `Assemble Final Email` → `Send Email (Gmail)`

#### 1.7 Manual Follow-up Task Creation

- **Overview:** For leads without websites, creates a Google Task to prompt manual follow-up call.
- **Nodes Involved:** `Set Task Data`, `Create Google Task for Follow-up`
- **Node Details:**

  - `Set Task Data`:
    - **Type:** Set Node
    - **Role:** Prepares task details including phone number, company name, address, and type.
  
  - `Create Google Task for Follow-up`:
    - **Type:** Google Tasks Node
    - **Role:** Creates a task in the user's selected Google Task list.
    - **Title:** "Call [Company Name] at [Phone Number] for website creation".
    - **Credentials:** Google Tasks OAuth2 credentials required.
    - **Failure Modes:** Authentication failure, invalid task list selection.

- **Connections:**
  - `If Website Exists` False → `Set Task Data`
  - `Set Task Data` → `Create Google Task for Follow-up`

#### 1.8 Documentation & Credentials Setup

- **Overview:** Sticky notes provide user instructions on setup, customization, and workflow logic.
- **Nodes Involved:** `Sticky Note`, `Sticky Note1`, `Sticky Note2`, `Sticky Note3`
- **Node Details:**

  - **Sticky Note (Setup Instructions):**
    - Lists all credentials needed (RapidAPI, Hunter.io, OpenAI, Gmail, Google Tasks).
    - Explains how to start the workflow using the form.
  
  - **Sticky Note1 (Customization):**
    - Guides on modifying AI prompt system message with company details.
    - Instructions for editing email signature in the email assembly node.
  
  - **Sticky Note2 (Workflow Logic Overview):**
    - Summarizes the workflow steps and decision points.
  
  - **Sticky Note3 (Manual Follow-up Configuration):**
    - Reminds user to connect Google account and select task list for manual follow-ups.

---

### 3. Summary Table

| Node Name                   | Node Type                  | Functional Role                          | Input Node(s)                  | Output Node(s)                     | Sticky Note                                                                                                  |
|-----------------------------|----------------------------|----------------------------------------|-------------------------------|----------------------------------|--------------------------------------------------------------------------------------------------------------|
| compagny_search             | Form Trigger               | Receive prospecting criteria input     | -                             | Search Local Businesses           |                                                                                                              |
| Search Local Businesses     | HTTP Request               | Fetch local businesses via RapidAPI    | compagny_search               | Split Out Businesses              |                                                                                                              |
| Split Out Businesses        | Split Out                  | Split business list into single items  | Search Local Businesses        | If Website Exists                |                                                                                                              |
| If Website Exists           | If                         | Check for website existence             | Split Out Businesses           | Scrape Website, Set Task Data     |                                                                                                              |
| Scrape Website             | HTTP Request               | Scrape company website HTML             | If Website Exists (true)       | If Scraping Succeeded             |                                                                                                              |
| If Scraping Succeeded       | If                         | Check scraping success                   | Scrape Website                 | Extract Title & Description       |                                                                                                              |
| Extract Title & Description | HTML Extraction            | Extract website title and description   | If Scraping Succeeded          | Find Contacts (Hunter)            |                                                                                                              |
| Find Contacts (Hunter)      | Hunter.io API              | Find company contacts by domain         | Extract Title & Description    | Split Out Contacts               |                                                                                                              |
| Split Out Contacts          | Split Out                  | Split contacts list into single items   | Find Contacts (Hunter)         | If Contact Is Valid              |                                                                                                              |
| If Contact Is Valid         | If                         | Validate contacts (name & verification) | Split Out Contacts             | Generate Email Body (AI)          |                                                                                                              |
| OpenAI Chat Model           | Langchain OpenAI Chat      | Provide GPT-5 model for AI generation   | -                             | Generate Email Body (AI) (AI model) |                                                                                                              |
| Generate Email Body (AI)    | Langchain Agent            | Generate personalized email body        | If Contact Is Valid, OpenAI Chat Model | Format AI Output             |                                                                                                              |
| Format AI Output            | Code                       | Format raw AI output into paragraphs     | Generate Email Body (AI)       | Assemble Final Email             |                                                                                                              |
| Assemble Final Email        | Set                        | Build email subject and body             | Format AI Output              | Send Email (Gmail)               |                                                                                                              |
| Send Email (Gmail)          | Gmail                      | Send email to contact                    | Assemble Final Email           | -                                |                                                                                                              |
| Set Task Data               | Set                        | Prepare task details for manual follow-up| If Website Exists (false)      | Create Google Task for Follow-up |                                                                                                              |
| Create Google Task for Follow-up | Google Tasks             | Create follow-up Google Task             | Set Task Data                 | -                                |                                                                                                              |
| Sticky Note                 | Sticky Note                | Setup instructions and credentials list | -                             | -                                | "## Quick Start & Required Setup\n\n**Action Required - Setup Credentials**:\nRapidAPI, Hunter.io, OpenAI, Gmail, Google Tasks" |
| Sticky Note1                | Sticky Note                | Customization instructions               | -                             | -                                | "## Customize Your Outreach\n\nEdit AI prompt and email signature for personalization."                      |
| Sticky Note2                | Sticky Note                | Workflow logic summary                    | -                             | -                                | "## Workflow Logic Overview\n\nSearch businesses, scrape sites, find contacts, generate and send emails."    |
| Sticky Note3                | Sticky Note                | Manual follow-up setup instructions      | -                             | -                                | "## Configure Manual Follow-up\n\nConnect Google account and select task list."                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node**  
   - Name: `compagny_search`  
   - Type: Form Trigger (version 2.2)  
   - Configure form fields:  
     - "Business Sector" (required, text input, placeholder "[ engineering ]")  
     - "city" (optional, text input, placeholder "[ London ]")  
     - "Number of Leads" (optional, text input, placeholder "[ 20 ]")  
   - This node starts the workflow on form submission.

2. **Add HTTP Request Node - Search Local Businesses**  
   - Name: `Search Local Businesses`  
   - Type: HTTP Request (version 4.2)  
   - URL: `https://local-business-data.p.rapidapi.com/search`  
   - Method: GET  
   - Query parameters:  
     - `query`: `={{ $json["Business Sector"] }} {{ $json.city }}`  
     - `limit`: `={{ $json["Number of Leads"] }}`  
     - `zoom`: `13`  
     - `language`: `fr`  
     - `region`: `fr`  
     - `extract_emails_and_contacts`: `false`  
   - Headers:  
     - `x-rapidapi-host`: `local-business-data.p.rapidapi.com`  
     - Add header auth credential with your RapidAPI key.  
   - Connect output of `compagny_search` to this node.

3. **Add Split Out Node - Split Out Businesses**  
   - Name: `Split Out Businesses`  
   - Type: Split Out (version 1)  
   - Field to split out: `data`  
   - Connect output of `Search Local Businesses` to this node.

4. **Add If Node - If Website Exists**  
   - Name: `If Website Exists`  
   - Type: If (version 2.2)  
   - Condition: Check if `$json.website` is not empty  
   - Connect output of `Split Out Businesses` to this node.

5. **Add HTTP Request Node - Scrape Website**  
   - Name: `Scrape Website`  
   - Type: HTTP Request (version 4.2)  
   - URL: `={{ $json.website }}`  
   - Method: GET  
   - On error: Continue regular output (to prevent workflow stop on failures)  
   - Connect "True" output of `If Website Exists` to this node.

6. **Add If Node - If Scraping Succeeded**  
   - Name: `If Scraping Succeeded`  
   - Type: If (version 2.2)  
   - Condition: Check that response JSON does not have `error` property  
   - Connect output of `Scrape Website` to this node.

7. **Add HTML Extraction Node - Extract Title & Description**  
   - Name: `Extract Title & Description`  
   - Type: HTML Extraction (version 1.2)  
   - Extraction values:  
     - Key: `scraped_title`, CSS selector: `title`, attribute: (none)  
     - Key: `scraped_description`, CSS selector: `meta[name="description"]`, attribute: `content`  
   - Connect "True" output of `If Scraping Succeeded` to this node.

8. **Add Hunter.io Node - Find Contacts (Hunter)**  
   - Name: `Find Contacts (Hunter)`  
   - Type: Hunter.io (version 1)  
   - Domain: `={{ $('Split Out Businesses').item.json.tld }}`  
   - Limit: 10 contacts  
   - Add Hunter.io API credentials.  
   - Connect output of `Extract Title & Description` to this node.

9. **Add Split Out Node - Split Out Contacts**  
   - Name: `Split Out Contacts`  
   - Type: Split Out (version 1)  
   - Field to split out: `value`  
   - Include all other fields.  
   - Connect output of `Find Contacts (Hunter)` to this node.

10. **Add If Node - If Contact Is Valid**  
    - Name: `If Contact Is Valid`  
    - Type: If (version 2.2)  
    - Conditions:  
      - `$json.first_name` is not empty  
      - `$('Find Contacts (Hunter)').item.json.verification.status` equals `"valid"`  
    - Connect output of `Split Out Contacts` to this node.

11. **Add Langchain OpenAI Chat Model Node**  
    - Name: `OpenAI Chat Model`  
    - Type: Langchain OpenAI Chat (version 1.2)  
    - Model: `gpt-5-nano-2025-08-07`  
    - Add OpenAI API credentials.  

12. **Add Langchain Agent Node - Generate Email Body (AI)**  
    - Name: `Generate Email Body (AI)`  
    - Type: Langchain Agent (version 2)  
    - Text input (prompt):  
      ```
      Here is the information about your target :

      - Contact firstname : {{ $json.first_name }}
      - Jobtittle : {{ $json.position }}
      - Compagny name: {{ $('If Website Exists').item.json.name }}
      - Website Tittle : "{{ $('Extract Title & Description').item.json.scraped_title }}"
      - Website Description : "{{ $('Extract Title & Description').item.json.scraped_description }}"
      ```
    - System message:  
      ```
      You are an expert in B2B sales prospecting, writing impactful outreach emails for the company [ Your_Compagny ] ([YOUR_COMPAGNY_WEBSITE]). 

      Your mission: Write a short (about 100 words), friendly, and professional email. The goal is to generate interest for a potential project in automation, technical SEO, and customer acquisition campaigns. 
      Start with a personalized hook based on their website's title or description. Show you've done your research. 
      Address the person according to their job title. 

      End with a simple, open-ended question to encourage a reply. The output must ONLY be the email body in plain text. Do NOT start with 'Hello [First Name],' and do NOT sign off at the end.
      ```
    - Connect "True" output of `If Contact Is Valid` and AI model node `OpenAI Chat Model` to this node.

13. **Add Code Node - Format AI Output**  
    - Name: `Format AI Output`  
    - Type: Code (version 2)  
    - JavaScript code:  
      ```js
      const items = $items();
      const newItems = items.map(item => {
        const json = item.json;
        const rawText = json.output;
        let formattedText = "";
        if (rawText && typeof rawText === 'string') {
          const sentences = rawText.match(/[^.!?]+[.!?]*/g) || [];
          formattedText = sentences
            .map(s => s.trim())
            .filter(s => s.length > 0)
            .join('\n\n');
        }
        return { json: { ...json, formatted_text: formattedText } };
      });
      return newItems;
      ```
    - Connect output of `Generate Email Body (AI)` to this node.

14. **Add Set Node - Assemble Final Email**  
    - Name: `Assemble Final Email`  
    - Type: Set (version 3.4)  
    - Assign:  
      - `email_subject`: `Developing the business of {{ $('If Website Exists').item.json.name }}`  
      - `email_body`:  
        ```
        Hi {{ $('Split Out Contacts').item.json.first_name }},  

        {{ $json.formatted_text }}

        Dear,
        ```
    - Connect output of `Format AI Output` to this node.

15. **Add Gmail Node - Send Email (Gmail)**  
    - Name: `Send Email (Gmail)`  
    - Type: Gmail (version 2.1)  
    - Send To: `={{ $('Find Contacts (Hunter)').item.json.value }}` (contact email)  
    - Subject: `={{ $json.email_subject }}`  
    - Message: `={{ $json.email_body }}`  
    - Add Gmail OAuth2 credentials.  
    - Connect output of `Assemble Final Email` to this node.

16. **Add Set Node - Set Task Data**  
    - Name: `Set Task Data`  
    - Type: Set (version 3.4)  
    - Assign:  
      - `phone_number`: `={{ $json.phone_number }}`  
      - `compagny_name`: `={{ $json.name }}`  
      - `adress`: `={{ $json.full_address }}`  
      - `type`: `={{ $json.type }}`  
    - Connect "False" output of `If Website Exists` to this node.

17. **Add Google Tasks Node - Create Google Task for Follow-up**  
    - Name: `Create Google Task for Follow-up`  
    - Type: Google Tasks (version 1)  
    - Task List: Select your Google Task list from dropdown  
    - Title: `=Appeler {{ $json.compagny_name }} au {{ $json.phone_number }} Pour création de site web !`  
    - Add Google Tasks OAuth2 credentials.  
    - Connect output of `Set Task Data` to this node.

18. **Add Sticky Notes for User Guidance**  
    - Add 4 sticky notes with the following content:  
      - Setup instructions and required credentials for RapidAPI, Hunter.io, OpenAI, Gmail, Google Tasks.  
      - Customization instructions for AI prompt and email signature.  
      - Workflow overview explaining logic flow.  
      - Instructions for configuring Google Tasks for manual follow-up.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                      | Context or Link                                                                                  |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| This workflow requires setting up various API credentials: RapidAPI for local business data, Hunter.io for contact discovery, OpenAI for AI email generation, Gmail for sending emails, and Google Tasks for manual follow-ups.                                                                                                                                                                                     | Credentials setup detailed in Sticky Note within workflow.                                      |
| Customize your AI prompt system message in the "Generate Email Body (AI)" node to reflect your company’s branding, services, and tone to maximize outreach effectiveness. Also, edit the "Assemble Final Email" node to add your professional email signature.                                                                                                                                                  | See Sticky Note1 for customization instructions.                                               |
| The workflow includes robust error handling: website scraping errors are caught and do not stop the flow; contacts are filtered for validity before AI generation; manual tasks are created for leads without websites to ensure no prospects are missed.                                                                                                                                                         | See Sticky Note2 and Sticky Note3 for workflow logic and manual follow-up details.             |
| For detailed API limits and best practices, consult the documentation of each service: RapidAPI Local Business Data, Hunter.io API, OpenAI API, Gmail API, and Google Tasks API.                                                                                                                                                                                                                                   | Official documentation links available on respective service websites.                         |
| Workflow designed to be triggered via a web form, ideal for sales teams or marketing automation professionals targeting local businesses in a specific sector and region.                                                                                                                                                                                                                                        |                                                                                               |

---

**Disclaimer:** The text provided originates exclusively from an n8n automated workflow. This process strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled are legal and publicly available.