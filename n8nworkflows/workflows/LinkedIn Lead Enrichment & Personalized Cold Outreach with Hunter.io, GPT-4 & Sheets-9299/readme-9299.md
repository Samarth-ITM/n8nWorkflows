LinkedIn Lead Enrichment & Personalized Cold Outreach with Hunter.io, GPT-4 & Sheets

https://n8nworkflows.xyz/workflows/linkedin-lead-enrichment---personalized-cold-outreach-with-hunter-io--gpt-4---sheets-9299


# LinkedIn Lead Enrichment & Personalized Cold Outreach with Hunter.io, GPT-4 & Sheets

### 1. Workflow Overview

This workflow automates LinkedIn lead enrichment and personalized cold outreach messaging by combining data scraping, email discovery, AI-generated message personalization, and data storage.

**Target Use Cases:**  
- Sales teams and business development professionals automating lead research and outreach personalization.  
- Growth marketers aiming to scale personalized email campaigns efficiently.  

**Logical Blocks:**  
- **1.1 Campaign Variable Setup:** Define target audience criteria and variables for personalization.  
- **1.2 LinkedIn Data Extraction:** Scrape LinkedIn profile URLs to gather professional information.  
- **1.3 Email Discovery:** Use Hunter.io API to find verified work email addresses for leads.  
- **1.4 AI Personalization:** Generate customized cold outreach messages using GPT-4 based on enriched lead data.  
- **1.5 Data Storage:** Save collected lead info and personalized messages into Google Sheets for tracking and follow-up.  
- **1.6 Rate Limiting:** Implement delays to respect API and scraping rate limits.  

---

### 2. Block-by-Block Analysis

#### 2.1 Campaign Variable Setup

- **Overview:**  
  Initializes campaign-specific variables including target industry, role, company name, value proposition, and LinkedIn URLs to process. Centralizes configuration for easy updates.

- **Nodes Involved:**  
  - Set Campaign Variables

- **Node Details:**  
  - **Set Campaign Variables**  
    - Type: Set  
    - Role: Defines static JSON data that includes target criteria and initial LinkedIn URLs array.  
    - Configuration: Raw JSON mode containing keys like `target_industry`, `target_role`, `company_name`, `value_proposition`, and an array of LinkedIn profile URLs.  
    - Inputs: None (start node)  
    - Outputs: Passes JSON to next node for splitting URLs.  
    - Edge Cases: Missing or malformed URLs in the array; requires manual update for different campaigns.  
    - Version: 3.4  

#### 2.2 LinkedIn Data Extraction

- **Overview:**  
  Processes each LinkedIn URL individually by scraping the profile HTML, then parsing key professional details like name, title, company, and location.

- **Nodes Involved:**  
  - Split LinkedIn URLs  
  - Rate Limit Delay  
  - Scrape LinkedIn Profile  
  - Parse LinkedIn Data

- **Node Details:**  
  - **Split LinkedIn URLs**  
    - Type: SplitOut  
    - Role: Splits the array of LinkedIn URLs into separate items for sequential processing.  
    - Configuration: Splits on the field `linkedin_urls`.  
    - Input: Campaign variables node  
    - Output: Single LinkedIn URL per item  
    - Edge Cases: Empty or invalid URLs cause failures downstream.  
    - Version: 1  

  - **Rate Limit Delay**  
    - Type: Wait  
    - Role: Enforces a 2-second delay between profile scrapes to avoid hitting LinkedIn rate limits.  
    - Configuration: Wait 2 seconds per item.  
    - Input: Split LinkedIn URLs  
    - Output: Scrape LinkedIn Profile  
    - Edge Cases: Too short delays risk IP blocking or captchas.  
    - Version: 1.1  

  - **Scrape LinkedIn Profile**  
    - Type: HTTP Request  
    - Role: Fetches LinkedIn profile HTML content using the URL.  
    - Configuration: HTTP GET request with custom User-Agent header to mimic a browser; 10-second timeout; uses generic HTTP header authentication (no auth token here).  
    - Input: Rate Limit Delay  
    - Output: Raw HTML content  
    - Edge Cases: HTTP failures, captchas, LinkedIn blocking scraping, invalid URLs.  
    - Version: 4.2  

  - **Parse LinkedIn Data**  
    - Type: Code  
    - Role: Extracts profile fields (full name, first/last name, title, company, location) from HTML using regex.  
    - Configuration: JavaScript code with helper function `extractText` for regex extraction on HTML snippets; outputs trimmed and cleaned data.  
    - Input: Scrape LinkedIn Profile  
    - Output: Parsed lead data JSON  
    - Edge Cases: HTML structure changes breaking regex; missing fields default to 'Unknown' or 'Not specified'.  
    - Version: 2  

#### 2.3 Email Discovery

- **Overview:**  
  Attempts to find verified professional email addresses using Hunter.io API based on parsed profile data. Marks if an email was found and its confidence score.

- **Nodes Involved:**  
  - Find Email with Hunter.io  
  - Process Email Result  
  - Email Found? (Conditional)  

- **Node Details:**  
  - **Find Email with Hunter.io**  
    - Type: HTTP Request  
    - Role: Queries Hunter.io email finder endpoint with domain (deduced from company name), first and last name.  
    - Configuration: GET request with query parameters including domain (normalized company name), first_name, last_name; uses query authentication with Hunter.io API key.  
    - Input: Parse LinkedIn Data  
    - Output: Hunter.io API JSON response  
    - Edge Cases: API rate limits, invalid domain extraction, no email found returns empty data, failures are tolerated by neverError=true.  
    - Version: 4.2  

  - **Process Email Result**  
    - Type: Code  
    - Role: Extracts email and confidence score from Hunter.io response; merges with lead data.  
    - Configuration: JavaScript code that reads API data and appends `email`, `email_confidence`, and `email_verified` flags.  
    - Input: Find Email with Hunter.io  
    - Output: Enhanced lead JSON with email info  
    - Edge Cases: Missing or null email fields; confidence fallback to zero.  
    - Version: 2  

  - **Email Found?**  
    - Type: If  
    - Role: Branches workflow depending on whether an email was found.  
    - Configuration: Checks if the `email` field is a non-empty string.  
    - Input: Process Email Result  
    - Output: True branch (email found) → AI Personalization; False branch → No Email - Skip.  
    - Edge Cases: Empty string or null may cause false negatives.  
    - Version: 2.2  

#### 2.4 AI Personalization

- **Overview:**  
  Uses GPT-4 Turbo AI to generate a custom cold outreach message tailored to the lead’s profile and campaign value proposition.

- **Nodes Involved:**  
  - Generate AI Personalization  
  - Merge AI Message  
  - No Email - Skip (alternative path)

- **Node Details:**  
  - **Generate AI Personalization**  
    - Type: OpenAI (LangChain)  
    - Role: Sends prompt with lead and campaign details to generate a personalized message.  
    - Configuration: Model `gpt-4-turbo-preview`; system prompt instructs to create a 100-word maximum, professional but conversational message based on input fields; outputs JSON with `message`.  
    - Input: Email Found? (true branch)  
    - Output: AI-generated message JSON  
    - Edge Cases: API failures, rate limits, prompt failures, or empty responses.  
    - Credential: OpenAI API key configured.  
    - Version: 1.8  

  - **Merge AI Message**  
    - Type: Code  
    - Role: Combines AI message with existing lead data; adds timestamp.  
    - Configuration: JavaScript merging previous JSON with AI response; fallback message if AI output missing.  
    - Input: Generate AI Personalization  
    - Output: Combined enriched lead data with personalized message field `ai_message`.  
    - Edge Cases: Missing AI message content; manual review may be needed.  
    - Version: 2  

  - **No Email - Skip**  
    - Type: NoOp  
    - Role: Terminates processing for leads without found emails gracefully.  
    - Input: Email Found? (false branch)  
    - Output: None (terminates this lead’s flow)  
    - Version: 1  

#### 2.5 Data Storage

- **Overview:**  
  Appends the enriched lead data, including personalized message and metadata, to a designated Google Sheet for campaign management.

- **Nodes Involved:**  
  - Save to Google Sheets

- **Node Details:**  
  - **Save to Google Sheets**  
    - Type: Google Sheets  
    - Role: Appends a row to the specified sheet with mapped columns matching lead and AI data fields.  
    - Configuration: Append operation; maps fields like Email, Title, Company, Location, Full Name, LinkedIn URL, AI message, email confidence, and current timestamp; uses Google Sheets document ID and sheet GID.  
    - Input: Merge AI Message  
    - Output: None (end of flow)  
    - Credential: Google Sheets OAuth2 credentials required.  
    - Edge Cases: Sheet access issues, mapping errors, API quota limits.  
    - Version: 4.5  

---

### 3. Summary Table

| Node Name               | Node Type                  | Functional Role                         | Input Node(s)             | Output Node(s)                 | Sticky Note                                                                                 |
|-------------------------|----------------------------|---------------------------------------|---------------------------|-------------------------------|---------------------------------------------------------------------------------------------|
| 📋 Workflow Overview & Setup Guide | Sticky Note                | Workflow description and setup guide  | —                         | —                             | 🎯 LinkedIn Lead Enrichment with AI Personalization; setup instructions and requirements    |
| Step 1 Context          | Sticky Note                | Explains campaign configuration       | —                         | —                             | Step 1: Configure Your Campaign; easy updates without editing multiple nodes                |
| Step 2 Context          | Sticky Note                | Explains LinkedIn scraping step       | —                         | —                             | Step 2: Extract LinkedIn Data; max 20 profiles/min rate limit                              |
| Step 3 Context          | Sticky Note                | Explains email discovery step          | —                         | —                             | Step 3: Find Email Addresses with Hunter.io; fallback logic                                |
| Step 4 Context          | Sticky Note                | Explains AI personalization step       | —                         | —                             | Step 4: AI Personalization based on prospect data and value proposition                     |
| Step 5 Context          | Sticky Note                | Explains data storage step             | —                         | —                             | Step 5: Store Enriched Leads in Google Sheets                                              |
| Set Campaign Variables  | Set                        | Defines target criteria and variables | —                         | Split LinkedIn URLs            |                                                                                             |
| Split LinkedIn URLs     | SplitOut                   | Splits LinkedIn URLs array into items | Set Campaign Variables     | Rate Limit Delay               |                                                                                             |
| Rate Limit Delay        | Wait                       | Enforces delay between scrapes        | Split LinkedIn URLs        | Scrape LinkedIn Profile        |                                                                                             |
| Scrape LinkedIn Profile | HTTP Request               | Scrapes LinkedIn profile HTML         | Rate Limit Delay           | Parse LinkedIn Data            |                                                                                             |
| Parse LinkedIn Data     | Code                       | Parses profile info from HTML         | Scrape LinkedIn Profile    | Find Email with Hunter.io      |                                                                                             |
| Find Email with Hunter.io | HTTP Request             | Queries Hunter.io for email addresses | Parse LinkedIn Data        | Process Email Result           |                                                                                             |
| Process Email Result    | Code                       | Extracts email info and merges data   | Find Email with Hunter.io  | Email Found?                   |                                                                                             |
| Email Found?            | If                         | Branches based on email presence      | Process Email Result       | Generate AI Personalization / No Email - Skip |                                                                                             |
| Generate AI Personalization | OpenAI (LangChain)       | Generates outreach message with GPT-4 | Email Found? (true branch) | Merge AI Message              |                                                                                             |
| Merge AI Message        | Code                       | Combines AI message with lead data    | Generate AI Personalization | Save to Google Sheets         |                                                                                             |
| No Email - Skip         | NoOp                       | Terminates flow if no email found     | Email Found? (false branch) | —                            |                                                                                             |
| Save to Google Sheets   | Google Sheets              | Appends enriched lead data            | Merge AI Message           | —                             |                                                                                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Set node named "Set Campaign Variables":**  
   - Mode: Raw JSON  
   - Define JSON with keys:  
     - `target_industry`: e.g. "SaaS"  
     - `target_role`: e.g. "Head of Sales"  
     - `company_name`: Your company name  
     - `value_proposition`: e.g. "We help sales teams book 30% more meetings using AI-powered personalization"  
     - `linkedin_urls`: Array of LinkedIn profile URLs to process  

3. **Add a SplitOut node named "Split LinkedIn URLs":**  
   - Field to split: `linkedin_urls`  
   - Connect "Set Campaign Variables" output to this node.

4. **Add a Wait node named "Rate Limit Delay":**  
   - Unit: seconds  
   - Amount: 2 seconds  
   - Connect "Split LinkedIn URLs" output to this node.

5. **Add an HTTP Request node named "Scrape LinkedIn Profile":**  
   - HTTP Method: GET  
   - URL: Expression `{{$json.linkedin_urls}}`  
   - Timeout: 10000 ms  
   - Set header `User-Agent` to mimic a browser (e.g., "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36")  
   - Authentication: None or generic HTTP header (if needed)  
   - Connect "Rate Limit Delay" output to this node.

6. **Add a Code node named "Parse LinkedIn Data":**  
   - Use JavaScript code to parse HTML content from the previous node: extract full name, first and last names, current title, company, location from LinkedIn profile HTML using regex.  
   - Include fallback values such as "Unknown" or "Not specified".  
   - Connect "Scrape LinkedIn Profile" output to this node.

7. **Add an HTTP Request node named "Find Email with Hunter.io":**  
   - HTTP Method: GET  
   - URL: `https://api.hunter.io/v2/email-finder`  
   - Query Parameters:  
     - `domain`: company name lowercased and sanitized + ".com"  
     - `first_name`: extracted first name  
     - `last_name`: extracted last name  
   - Authentication: Set Hunter.io API key credential (queryAuth)  
   - Configure to never error on failure to gracefully handle no results.  
   - Connect "Parse LinkedIn Data" output to this node.

8. **Add a Code node named "Process Email Result":**  
   - Extract email and score from Hunter.io response; merge with lead data.  
   - Add `email_verified` boolean flag based on presence of email.  
   - Connect "Find Email with Hunter.io" output to this node.

9. **Add an If node named "Email Found?":**  
   - Condition: Check if `email` field is not empty string.  
   - Connect "Process Email Result" output to this node.

10. **Add an OpenAI node named "Generate AI Personalization":**  
    - Model: GPT-4 Turbo Preview  
    - Prompt: Provide a system prompt that instructs the model to generate a personalized cold outreach message under 100 words, referencing lead's name, title, company, and campaign value proposition.  
    - Use expressions to plug lead data and campaign variables into prompt.  
    - Connect "Email Found?" true output to this node.  
    - Configure OpenAI API key credential.

11. **Add a Code node named "Merge AI Message":**  
    - Merge AI response message with existing lead data.  
    - Add timestamp field for processing time.  
    - Connect "Generate AI Personalization" output to this node.

12. **Add a Google Sheets node named "Save to Google Sheets":**  
    - Operation: Append  
    - Document ID: Set your Google Sheet ID  
    - Sheet Name: The GID or sheet name to append to  
    - Map columns to lead fields: Email, Title, Company, Location, Full Name, Last Name, First Name, LinkedIn URL, Enriched Date (timestamp), Email Confidence, Personalized Message  
    - Connect "Merge AI Message" output to this node.  
    - Configure Google Sheets OAuth2 credentials.

13. **Add a NoOp node named "No Email - Skip":**  
    - Connect "Email Found?" false output here to halt processing leads without emails.

14. **Link all nodes according to the above connections to ensure proper flow.**

15. **Add Sticky Notes at relevant positions describing each step as per the workflow overview and context notes for clarity and maintenance.**

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                              |
|---------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------|
| Workflow uses community nodes for LinkedIn scraping; self-hosted n8n instance required due to scraping limits. | See sticky note: Important for scraping LinkedIn profiles.  |
| Setup Guide Video placeholder: Add your Loom video link here to assist users visually.                         | Located in "📋 Workflow Overview & Setup Guide" sticky note. |
| Pro Tip: Use the Set Fields node to modify target industry and value proposition easily without editing many nodes. | Mentioned in overview note for efficient customization.      |

---

**Disclaimer:**  
The provided text is exclusively derived from an automated n8n workflow. It complies strictly with current content policies and contains no illegal or protected content. All processed data is legal and publicly available.